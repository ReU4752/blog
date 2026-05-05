+++
date = '2026-05-05T18:13:15+09:00'
draft = false
title = 'Dangling Pointer를 피하는 Weak Pointer 구현 방식 정리'
tags = ['memory','weakptr']
categories = ['']
+++

## 서론

최근 UnrealEngine 의 WeakObjectPtr 의 내부 구현을 보다가 ObjectSerialNumber 라는 멤버를 확인했다.

간략하게 `ObjectSerialNumber`는 weak pointer가 가리키던 UObject 가 아직 같은 객체인지 확인하기 위한 serial 값이다.
`TWeakObjectPtr`가 UObject를 가리키도록 설정될 때 해당 object array slot에 serial이 아직 없으면, 전역 counter를 1 증가시켜 lazy 하게 발급한다.
실제 식별은 `ObjectIndex + ObjectSerialNumber` 조합으로 한다.

여기서 궁금했던 것은 ObjectSerialNumber 가 int32 인데 int32 최대값인 2^31-1 = 2,147,483,647 를 넘어가면 충돌의 위험이 있을 수 있지 않을까? 라고 생각했다.

신기했던 점은 엔진은 overflow 발생 시 fatal 로그를 남기고 있다. "UObject serial numbers overflowed"
아마 엔진은 프로세스 생명주기 동안 "약 21억 개 수준의 serial 할당 전에 끝난다"는 전제로 보인다.
관련 흐름만 줄이면 다음과 같다.
```cpp
int32 FUObjectArray::AllocateSerialNumber(int32 Index)
{
    FUObjectItem* item = IndexToObject(Index);
    volatile int32* serialPtr = &item->SerialNumber;

    int32 serial = *serialPtr;
    if (serial != 0)
        return serial;

    serial = MasterSerialNumber.Increment();

    UE_CLOG(serial <= START_SERIAL_NUMBER, LogUObjectArray, Fatal,
        TEXT("UObject serial numbers overflowed "
             "(trying to allocate serial number %d)."), serial);

    int32 previous = FPlatformAtomics::InterlockedCompareExchange(
        (int32*)serialPtr, serial, 0);

    return previous != 0 ? previous : serial;
}
```

하지만 분명 21억 이상의 serial 발급이 필요한 상황 혹은 프로그램이 있을 수 있고 다른 weak 구현 방식은 없을까라고 생각했었고, 찾은 내용들을 정리해봤다.


## 여러 Weak ptr 구현 방식

대표적인 방식은 아래와 같다.

1. Index + Serial / Generation Handle
    * Unreal `TWeakObjectPtr`, ECS handle
2. Control Block 기반 weak
    * `std::weak_ptr`, Unreal `TWeakPtr`, Rust `Arc::Weak`
3. GC Weak Reference Table
    * Java `WeakReference`, C# `WeakReference`
4. Zeroing Weak Runtime
    * Objective-C ARC `__weak`, Swift `weak`

예시 코드는 읽기 쉽도록 핵심 흐름만 남겼다.

### Index + Serial / Generation Handle

아래 코드는 개념을 보여주기 위한 단일 스레드 예제다.
실제 엔진에서는 object table의 동기화, slot 재사용 정책, serial overflow 처리가 함께 필요하다.

```cpp
template <class T>
class ObjectPool
{
public:
    struct WeakHandle
    {
        std::size_t index = static_cast<std::size_t>(-1);
        std::uint32_t generation = 0;
    };

    ~ObjectPool()
    {
        for (Slot& slot : slots_)
        {
            delete slot.ptr;
        }
    }

    template <class... Args>
    WeakHandle create(Args&&... args)
    {
        std::size_t index = 0;
        while (index < slots_.size() && slots_[index].alive)
            ++index;

        if (index == slots_.size())
            slots_.push_back({});

        Slot& slot = slots_[index];
        slot.ptr = new T(std::forward<Args>(args)...);
        slot.alive = true;

        return {index, slot.generation};
    }

    void destroy(WeakHandle handle)
    {
        if (!isLive(handle))
            return;

        Slot& slot = slots_[handle.index];
        delete slot.ptr;
        slot.ptr = nullptr;
        slot.alive = false;

        ++slot.generation;

        if (slot.generation == 0)
            std::terminate();
    }

    T* get(WeakHandle handle) const
    {
        return isLive(handle) ? slots_[handle.index].ptr : nullptr;
    }

private:
    struct Slot
    {
        T* ptr = nullptr;
        std::uint32_t generation = 1;
        bool alive = false;
    };

    bool isLive(WeakHandle handle) const
    {
        if (handle.index >= slots_.size())
            return false;

        const Slot& slot = slots_[handle.index];
        return slot.alive && slot.generation == handle.generation;
    }

    std::vector<Slot> slots_;
};
```

핵심 구조
```
WeakHandle = index + generation
```

index가 같아도 generation이 다르면 예전 객체로 판단해서 invalid 처리.

### Control Block 기반 weak

아래 구현은 control block과 ref count 구조를 보여주기 위한 단일 스레드 예제다.
실제 `std::weak_ptr::lock()`이나 Rust `Weak::upgrade()`는 다른 스레드의 strong owner 파괴와 경쟁할 수 있으므로, strong count 확인과 증가가 원자적으로 처리되어야 한다.

```cpp
template <class T>
struct ControlBlock
{
    T* ptr = nullptr;
    std::size_t strongCount = 1;
    std::size_t weakCount = 0;
};

template <class T>
class WeakPtr;

template <class T>
class SharedPtr
{
public:
    explicit SharedPtr(T* ptr)
        : control_(ptr ? new ControlBlock<T>{ptr} : nullptr)
    {
    }

    SharedPtr(const SharedPtr& other)
        : control_(other.control_)
    {
        if (control_)
            ++control_->strongCount;
    }

    ~SharedPtr()
    {
        release();
    }

    SharedPtr& operator=(SharedPtr other)
    {
        std::swap(control_, other.control_);
        return *this;
    }

    T* get() const
    {
        return control_ ? control_->ptr : nullptr;
    }

private:
    friend class WeakPtr<T>;

    struct AddStrong {};

    SharedPtr(ControlBlock<T>* control, AddStrong)
        : control_(control)
    {
        if (control_)
            ++control_->strongCount;
    }

    void release()
    {
        if (!control_)
            return;

        if (--control_->strongCount == 0)
        {
            delete control_->ptr;
            control_->ptr = nullptr;

            if (control_->weakCount == 0)
                delete control_;
        }

        control_ = nullptr;
    }

private:
    ControlBlock<T>* control_ = nullptr;
};

template <class T>
class WeakPtr
{
public:
    explicit WeakPtr(const SharedPtr<T>& shared)
        : control_(shared.control_)
    {
        if (control_)
            ++control_->weakCount;
    }

    WeakPtr(const WeakPtr& other)
        : control_(other.control_)
    {
        if (control_)
            ++control_->weakCount;
    }

    ~WeakPtr()
    {
        release();
    }

    WeakPtr& operator=(WeakPtr other)
    {
        std::swap(control_, other.control_);
        return *this;
    }

    SharedPtr<T> lock() const
    {
        if (!control_ || control_->strongCount == 0)
            return SharedPtr<T>(nullptr);

        return SharedPtr<T>(control_, typename SharedPtr<T>::AddStrong{});
    }

private:
    void release()
    {
        if (!control_)
            return;

        --control_->weakCount;

        if (control_->weakCount == 0 && control_->strongCount == 0)
            delete control_;

        control_ = nullptr;
    }

private:
    ControlBlock<T>* control_ = nullptr;
};
```

핵심 구조
```
WeakPtr
{
    storedPtr;      // 구현에 따라 생략되거나 control block 안의 ptr을 쓸 수 있음
    controlBlock*;
}

ControlBlock
{
    managedPtr;
    strongCount;
    weakCount;
}
```

객체는 strongCount == 0일 때 삭제.
하지만 WeakPtr이 남아 있으면 ControlBlock은 남아 있어서 expired 여부를 판단할 수 있다.

### GC Weak Reference Table

아래 코드는 weak reference table의 처리 순서를 보여주는 단순 mark-sweep 예제다.
mark/sweep의 세부 구현과 테스트용 객체 생성 코드는 생략했다.
실제 Java나 .NET에서는 finalization, reference queue, 세대별 GC 같은 런타임 정책 때문에 clear와 reclaim의 정확한 시점이 구현별로 달라질 수 있다.

```cpp
class GCObject
{
public:
    bool marked = false;
};

class WeakRefBase
{
public:
    virtual ~WeakRefBase() = default;
    virtual void clearIfUnmarked() = 0;
};

void GCRuntime::collect(const std::vector<GCObject*>& roots)
{
    clearMarks();
    markFrom(roots);

    // strong graph에서 도달 불가능한 객체를 가리키는 weak는 먼저 clear한다.
    for (WeakRefBase* weak : weakRefs_)
        weak->clearIfUnmarked();

    sweepUnmarked();
}

template <class T>
class WeakRef : public WeakRefBase
{
public:
    explicit WeakRef(T* ptr = nullptr)
        : ptr_(ptr)
    {
        GGC.addWeakRef(this);
    }

    ~WeakRef() override
    {
        GGC.removeWeakRef(this);
    }

    T* get() const
    {
        return ptr_;
    }

    void clearIfUnmarked() override
    {
        if (ptr_ && !ptr_->marked)
            ptr_ = nullptr;
    }

private:
    T* ptr_ = nullptr;
};
```

핵심 구조
```
GCRuntime
{
    objects;
    weakRefs;
}
```

GC가 mark 단계에서 살아남지 못한 객체를 찾고, weak 처리 phase에서 그 객체를 가리키는 weak reference를 먼저 clear한다.
이후 finalization/reclaim 시점은 런타임마다 다르며, 위 예제는 이해를 돕기 위해 clear 직후 delete하는 단순 모델이다.

### Zeroing Weak Runtime

아래 코드는 zeroing weak의 동작을 C++로 흉내 낸 개념 예제다. 간단하게 등록/해제/zeroing 흐름만 남겼다.
Objective-C/Swift의 실제 런타임은 객체가 직접 `vector`를 들고 있기보다는 runtime side table 또는 weak registry를 통해 weak slot들을 추적한다고 보는 편이 더 정확하다.

```cpp
class ZeroingWeakBase
{
public:
    virtual ~ZeroingWeakBase() = default;
    virtual void zero() = 0;
};

class Zeroable
{
public:
    void addWeak(ZeroingWeakBase* weak)
    {
        weakRefs_.push_back(weak);
    }

    void removeWeak(ZeroingWeakBase* weak)
    {
        weakRefs_.erase(
            std::remove(weakRefs_.begin(), weakRefs_.end(), weak),
            weakRefs_.end()
        );
    }

    virtual ~Zeroable()
    {
        for (ZeroingWeakBase* weak : weakRefs_)
            weak->zero();
    }

private:
    std::vector<ZeroingWeakBase*> weakRefs_;
};

template <class T>
class ZeroingWeak : public ZeroingWeakBase
{
public:
    static_assert(
        std::is_base_of<Zeroable, T>::value,
        "T must derive from Zeroable"
    );

    explicit ZeroingWeak(T* ptr = nullptr)
    {
        reset(ptr);
    }

    ZeroingWeak(const ZeroingWeak&) = delete;
    ZeroingWeak& operator=(const ZeroingWeak&) = delete;

    ~ZeroingWeak() override
    {
        reset(nullptr);
    }

    void reset(T* ptr)
    {
        if (ptr_)
            ptr_->removeWeak(this);

        ptr_ = ptr;

        if (ptr_)
            ptr_->addWeak(this);
    }

    T* get() const
    {
        return ptr_;
    }

    void zero() override
    {
        // target은 이미 파괴 중이므로 removeWeak를 호출하지 않는다.
        ptr_ = nullptr;
    }

private:
    T* ptr_ = nullptr;
};
```

핵심 구조
```
RuntimeWeakRegistry / SideTable
{
    target -> weakSlots[]
}
```

개념적으로는 객체가 파괴될 때 자신을 가리키는 weak slot들을 찾아서 `nil`/`nullptr`로 바꾼다.
다만 위 C++ 예제는 `Zeroable` base destructor에서 zeroing하므로, derived destructor가 실행되는 동안에는 weak가 아직 non-null일 수 있다.
실제 런타임은 final release, weak load/store, weak clear 사이의 동기화까지 처리해야 한다.

### 요약

| 방식             | weak가 들고 있는 것                    | 무효화 시점                                |
| -------------- | -------------------------------- | ------------------------------------- |
| Index + Serial | `index + generation`             | `get()` 시 table의 generation과 비교       |
| Control Block  | `stored pointer`, `control block*` | `lock()` 시 strong count 확인            |
| GC Weak Table  | `target*`, runtime에 등록           | GC가 unreachable 객체를 발견했을 때            |
| Zeroing Weak   | `target*`, runtime weak registry에 등록 | target deallocation 시 weak slot들을 nil/null 처리 |


## 정리

방식별 장/단점을 요약한 표. 성능/메모리는 대표적인 구현 경향 기준이고, 실제 비용은 플랫폼/라이브러리/엔진 버전에 따라 달라질 수 있다.

| 방식                                     | 예시                                                   | 장점                                                                                            | 단점                                                                                          | 접근/Get 비용                                                                                                                                                   | 생성/복사/파괴 비용                                                                                                                                                                     | 메모리 사용량                                                                                                                                                                                                                           |
| -------------------------------------- | ---------------------------------------------------- | --------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Index + Serial / Generation Handle** | Unreal `TWeakObjectPtr`, ECS handle                  | weak 하나가 작고, 별도 allocation이 거의 없음. 객체가 move/compaction 되더라도 table index 기반이면 추적 가능. 대량 객체에 적합 | 전역 object table 필요. generation/serial overflow 이슈가 있음. 직접 포인터보다 한 번 더 우회                    | **낮음~중간**. `ObjectIndex`로 table 조회 후 serial 비교. raw pointer 직접 접근보다는 비쌈                                                                                     | **낮음**. weak 생성은 index/serial 저장. serial 최초 할당/slot 재사용 시 약간의 비용                                                                                                                | **weak당 낮음**. 보통 index + serial 수준. 대신 전역 object array / slot metadata가 필요                                                                                                                                                        |
| **Control Block 기반 weak**              | `std::weak_ptr`, Unreal `TWeakPtr`, Rust `Arc::Weak` | serial 충돌 없음. 객체가 삭제돼도 control block이 identity 역할을 유지. 일반 C++ 객체에 쓰기 좋음                       | 객체가 `shared_ptr`/`Arc` 같은 shared ownership으로 관리되어야 함. ref count 비용이 있음                      | **중간**. `lock()`/`upgrade()` 시 control block 확인 + strong count 증가 시도. C++ `weak_ptr::lock()`은 살아 있으면 `shared_ptr`를 반환하고, 아니면 빈 값을 반환한다. ([Cppreference][1]) | **중간**. weak 복사/파괴 시 weak count 증가/감소. 멀티스레드 `shared_ptr`/`Arc`면 보통 atomic 비용                                                                                                   | **weak당 중간**. weak 객체 자체는 보통 포인터 1~2개 수준이지만, control block이 별도로 필요. 마지막 strong owner가 사라져도 weak가 남아 있으면 control block/backing allocation이 남는다. ([Cppreference][2])                                                                |
| **GC Weak Reference Table**            | Java `WeakReference`, C# `WeakReference`             | GC가 객체 생존 여부를 알고 있어서 dangling pointer 문제가 없음. 캐시, canonical map에 적합                           | 수거 시점이 비결정적. GC 처리 비용에 영향을 줄 수 있음. 네이티브 즉시 파괴 모델과는 다름                                       | **낮음~중간**. weak reference 객체에서 target을 읽음. 단, GC 타이밍에 따라 null이 될 수 있음                                                                                       | **중간**. weak reference 객체 생성 + GC의 weak 처리/queue 처리 비용                                                                                                                          | **중간~높음**. weak reference 자체가 관리 객체이고, GC가 별도 추적해야 함. Java는 weakly reachable 객체에 대한 weak reference를 GC가 clear하고 필요하면 queue에 넣는다. ([Oracle Docs][3]) .NET도 strong reference가 없으면 GC가 대상 객체를 수거할 수 있다고 설명한다. ([Microsoft Learn][4]) |
| **Zeroing Weak Runtime**               | Objective-C ARC `__weak`, Swift `weak`               | 대상 객체가 파괴되면 weak slot이 자동으로 `nil` 처리됨. 사용자는 null check만 하면 됨                                  | runtime이 weak slot들을 추적해야 함. weak store/load/destroy에 런타임 비용이 있음. 객체 파괴 시 연결된 weak들을 정리해야 함 | **중간**. 단순 raw pointer보다 비쌀 수 있음. 특히 weak load를 안전하게 strongify하는 경우 비용 증가                                                                                   | **중간~높음**. weak 저장/해제 시 runtime 등록/해제. Clang ARC 문서는 runtime이 non-null `__weak` object를 추적하고 `objc_storeWeak`, `objc_destroyWeak`, `objc_moveWeak`를 통해 관리한다고 설명한다. ([Clang][5]) | **중간**. weak 변수 자체 저장공간은 포인터 수준이지만, runtime side table / weak registry가 필요                                                                                                                                                        |

[1]: https://en.cppreference.com/cpp/memory/weak_ptr/lock "std::weak_ptr<T>::lock"
[2]: https://en.cppreference.com/cpp/memory/shared_ptr "std::shared_ptr"
[3]: https://docs.oracle.com/javase/8/docs/api/java/lang/ref/WeakReference.html "WeakReference (Java Platform SE 8 )"
[4]: https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/weak-references "Weak References - .NET | Microsoft Learn"
[5]: https://clang.llvm.org/docs/AutomaticReferenceCounting.html "Objective-C Automatic Reference Counting (ARC) — Clang 23.0.0git documentation"
