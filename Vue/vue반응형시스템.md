## Vue3 반응형 시스템 소스코드 분석

### 용어

**effect**: 반응형 데이터가 변경될 때마다 자동으로 다시 실행되는 함수를 의미한다.

**track**: 종속성 추적을 의미한다. 어떤 데이터가 변경되었을 때 어떤 부분을 업데이트해야 하는지를 기억하기 위해 그 관계를 추적하고 기록하는 과정을 말한다.

**trigger**: 데이터가 변경되었을 때 dep에 저장된 모든 이펙트들을 실행하도록 신호를 보내는 동작을 의미한다.

**sub**: 구독자(Subscriber)를 의미한다. 반응형 데이터가 변경되었을 때 반응하는 대상이다. 즉, 반응형 상태에 의존하는 컴포넌트, effect와 같은 객체가 subscriber가 된다.

**dep**: 의존성(Dependency)을 의미한다. `ref()` 또는 `computed()`로 생성된 반응형 데이터를 말한다. Vue에서 사용하는 반응형 변수는 dep로 표현하는데, 이는 변경 사항을 감지하고, 연관된 subscriber에게 알림(trigger)을 준다.

> Dep는 데이터 그 자체, Sub는 데이터를 사용하는 로직이나 컴포넌트를 의미

**link**: dep와 sub 간의 연결을 나타낸다. dep는 여러개의 sub를 가질 수 있고, sub도 여러개의 dep와 연결될 수 있다. 이러한 n:m 관계를 효율적으로 관리하기 위해 중간 연결 개념으로 사용된다.

### Link 클래스

```ts
export class Link {
  version: number;

  nextDep?: Link;
  prevDep?: Link;
  nextSub?: Link;
  prevSub?: Link;
  prevActiveLink?: Link;

  constructor(public sub: Subscriber, public dep: Dep) {
    this.version = dep.version;
    this.nextDep =
      this.prevDep =
      this.nextSub =
      this.prevSub =
      this.prevActiveLink =
        undefined;
  }
}
```

하나의 Dep와 하나의 Subscriber(effect)를 연결하는 이중 연결 리스트이다.

### 반응형 데이터 클래스

```ts
/** ref 메서드가 반환하는 클래스 */
class RefImpl<T = any> {
  _value: T;
  private _rawValue: T;

  dep: Dep = new Dep();

  public readonly [ReactiveFlags.IS_REF] = true;
  public readonly [ReactiveFlags.IS_SHALLOW]: boolean = false;

  constructor(value: T, isShallow: boolean) {
    this._rawValue = isShallow ? value : toRaw(value);
    this._value = isShallow ? value : toReactive(value);
    this[ReactiveFlags.IS_SHALLOW] = isShallow;
  }

  get value() {
    this.dep.track();
    return this._value;
  }

  set value(newValue) {
    const oldValue = this._rawValue;
    const useDirectValue =
      this[ReactiveFlags.IS_SHALLOW] ||
      isShallow(newValue) ||
      isReadonly(newValue);
    newValue = useDirectValue ? newValue : toRaw(newValue);
    if (hasChanged(newValue, oldValue)) {
      this._rawValue = newValue;
      this._value = useDirectValue ? newValue : toReactive(newValue);
      this.dep.trigger();
    }
  }
}
```

ref 메서드가 반환하는 객체인 RefImpl 클래스를 살펴보면, value 값을 읽을 때 `this.dep.track()`을 호출하고 있고, value 값을 변경할 때에는 `this.dep.trigger()`를 호출하고 있다.

### Dep 클래스

```ts
export class Dep {
  version = 0;
  activeLink?: Link = undefined;
  subs?: Link = undefined;
  subsHead?: Link;
  map?: KeyToDepMap = undefined;
  key?: unknown = undefined;
  sc: number = 0;
  readonly __v_skip = true;

  track(debugInfo?: DebuggerEventExtraInfo): Link | undefined {
    if (!activeSub || !shouldTrack || activeSub === this.computed) {
      return;
    }

    let link = this.activeLink;
    if (link === undefined || link.sub !== activeSub) {
      link = this.activeLink = new Link(activeSub, this);

      if (!activeSub.deps) {
        activeSub.deps = activeSub.depsTail = link;
      } else {
        link.prevDep = activeSub.depsTail;
        activeSub.depsTail!.nextDep = link;
        activeSub.depsTail = link;
      }

      addSub(link);
    } else if (link.version === -1) {
      link.version = this.version;
      if (link.nextDep) {
        const next = link.nextDep;
        next.prevDep = link.prevDep;
        if (link.prevDep) {
          link.prevDep.nextDep = next;
        }

        link.prevDep = activeSub.depsTail;
        link.nextDep = undefined;
        activeSub.depsTail!.nextDep = link;
        activeSub.depsTail = link;

        if (activeSub.deps === link) {
          activeSub.deps = next;
        }
      }
    }

    return link;
  }

  trigger(debugInfo?: DebuggerEventExtraInfo): void {
    this.version++;
    globalVersion++;
    this.notify(debugInfo);
  }

  notify(debugInfo?: DebuggerEventExtraInfo): void {
    startBatch(); // 큐 초기화
    try {
      for (let link = this.subs; link; link = link.prevSub) {
        if (link.sub.notify()) {
          // notify()가 true를 반환하는 경우 computed 라는 의미이다.
          (link.sub as ComputedRefImpl).dep.notify();
        }
      }
    } finally {
      endBatch(); // 큐에 저장된 이펙트 한 번에 실행
    }
  }
}
```

**activeSub**: 현재 실행중인 이펙트(Subscriber)를 가르키는 전역 변수

**activeLink**: 현재 실행중인 이펙트와 연결된 Link

### Dep의 track 메서드

sub와 dep를 연결하는 Link를 생성하거나, 사용하지 않는 링크는 삭제하는 동작을 한다.

### Dep의 trigger 메서드

dep의 version을 업데이트 후 batch 방식으로 dep

**startBatch()**: 배치 모드를 시작하는 메서드이다. trigger 호출로 실행 대상이 된 이펙트들을 큐에 모아 기록한다.

**endBatch()**: 모아둔 작업을 한번에 실행하는 메서드이다. 배치 모드를 끝내고 큐에 모아둔 모든 이펙트를 중복 제거 후 순차적으로 실행한다.

### endBatch 메서드

```ts
export function endBatch(): void {
  if (--batchDepth > 0) {
    return;
  }

  if (batchedComputed) {
    let e: Subscriber | undefined = batchedComputed;
    batchedComputed = undefined;
    while (e) {
      const next: Subscriber | undefined = e.next;
      e.next = undefined;
      e.flags &= ~EffectFlags.NOTIFIED;
      e = next;
    }
  }

  let error: unknown;
  while (batchedSub) {
    let e: Subscriber | undefined = batchedSub;
    batchedSub = undefined;
    while (e) {
      const next: Subscriber | undefined = e.next;
      e.next = undefined;
      e.flags &= ~EffectFlags.NOTIFIED;
      if (e.flags & EffectFlags.ACTIVE) {
        try {
          // ACTIVE flag is effect-only
          (e as ReactiveEffect).trigger();
        } catch (err) {
          if (!error) error = err;
        }
      }
      e = next;
    }
  }

  if (error) throw error;
}
```

**batchedSub**: startBatch에서 배치된 이펙트들의 연결 리스트이다.

- batchedSub의 모든 이펙트의 trigger 메서드를 호출 후 비워버린다.

**ReactiveEffect**: effect로 관리되는 computed, watch, watchEffect, componenet 등을 사용할 때 생성되는 클래스이다.

### ReactiveEffect 클래스의 멤버 메서드

```ts
  trigger(): void {
    if (this.flags & EffectFlags.PAUSED) { // 일시 중단 상태라면?
      pausedQueueEffects.add(this)
    } else if (this.scheduler) { // 스케줄러가 존재한다면?
      this.scheduler()
    } else {
      this.runIfDirty()
    }
  }

  /**
   * @internal
   */
  runIfDirty(): void {
    if (isDirty(this)) {
      this.run()
    }
  }

  get dirty(): boolean {
    return isDirty(this)
  }

  run(): T {
    if (!(this.flags & EffectFlags.ACTIVE)) {
      return this.fn()
    }

    this.flags |= EffectFlags.RUNNING
    cleanupEffect(this)
    prepareDeps(this)
    const prevEffect = activeSub
    const prevShouldTrack = shouldTrack
    activeSub = this
    shouldTrack = true

    try {
      return this.fn()
    } finally {
      if (__DEV__ && activeSub !== this) {
        warn(
          'Active effect was not restored correctly - ' +
            'this is likely a Vue internal bug.',
        )
      }
      cleanupDeps(this)
      activeSub = prevEffect
      shouldTrack = prevShouldTrack
      this.flags &= ~EffectFlags.RUNNING
    }
  }
```

trigger는 상태가 변경되었다면 run 메서드를 실행하여 fn 함수를 호출하는데, 이 함수가 바로 watch, watchEffect, computed, component 등에서 사용하는 이펙트이다.

**flags**: 이펙트의 상태를 비트 플래그로 나타내는 정수값

핵심 플래그

- ACTIVE: 활성화된 이펙트
- NOTIFIED: 이번 배치에서 이미 큐에 추가된 이펙트 (중복 방지)
- PAUSED: 일시 중단 상태 (suspense, transition 와 같은 특수 상황)

**dirty**: 상태가 변경되었음을 의미

> 다음에 계속..
