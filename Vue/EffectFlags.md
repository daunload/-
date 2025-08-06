# EffectFlags

```typescript
export enum EffectFlags {
  /**
   * ReactiveEffect only
   */
  ACTIVE = 1 << 0,
  RUNNING = 1 << 1,
  TRACKING = 1 << 2,
  NOTIFIED = 1 << 3,
  DIRTY = 1 << 4,
  ALLOW_RECURSE = 1 << 5,
  PAUSED = 1 << 6,
  EVALUATED = 1 << 7,
}
```

Vue 반응형 시스템에서 Effect의 현재 상태를 효율적으로 관리하기 위해 사용되는 열거형이다.

하나의 숫자 값에 여러 개의 상태를 저장하기 위해 bitmask 기법을 사용한다.

bitmask 기법은 여러 개의 변수를 사용하는 것보다 메모리 효율적이고 빠른 처리를 가능하게 한다.

## EffectFlags 각 플래그의 역할

### Active

Effect가 의존성을 추적하고 실행될 수 있는 상태인지를 나타낸다.

```typescript

export class ReactiveEffect<T = any>
  implements Subscriber, ReactiveEffectOptions
{
  deps?: Link = undefined
  depsTail?: Link = undefined
  flags: EffectFlags = EffectFlags.ACTIVE | EffectFlags.TRACKING
  next?: Subscriber = undefined

  stop(): void {
    if (this.flags & EffectFlags.ACTIVE) {
      for (let link = this.deps; link; link = link.nextDep) {
        removeSub(link)
      }
      this.deps = this.depsTail = undefined
      cleanupEffect(this)
      this.onStop && this.onStop()
      this.flags &= ~EffectFlags.ACTIVE
    }
  }
```

ReactiveEffect 인스턴스가 생성될 때 기본값으로 설정된다. `stop()` 함수가 호출되면 이 플래그가 제거되며, `ACTIVE` 플래그가 없는 Effect는 더 이상 의존성 변경에 의해 트리거되지 않는다.

### RUNNING

Effect의 핵심 함수가 현재 실행중인지 나타낸다.

```typescript

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

이 플래그의 주된 목적은 무한 재귀 호출을 방지하는 것이다. Effect 함수가 실행되는 동안 자기 자신을 다시 트리거하는 경우를 막아준다.
`run()` 메서드 시작 시 설정되고, `finally` 블록에서 해제된다.
`notify()` 메서드는 이 플래그를 확인하여 이미 실행 중인 Effect가 다시 큐에 추가되는 것을 방지한다.

### TRACKING

Effect가 자신의 의존성을 추적해야 하는 상태인지를 나타낸다.

주로 `computed`와 관련이 깊다. `computed`는 기본적으로 lazy하게 동작하므로, 다른 Effect가 해당 `computed`를 구독하기 전까지는 의존성을 추적할 필요가 없다. 첫 구독자가 생기면 이 플래그가 활성화되어 의존성 추적을 시작하고, 구독자가 모두 사라지면 비활성화된다.

```typescript
function removeSub(link: Link, soft = false) {
  const { dep, prevSub, nextSub } = link;
  if (prevSub) {
    prevSub.nextSub = nextSub;
    link.prevSub = undefined;
  }
  if (nextSub) {
    nextSub.prevSub = prevSub;
    link.nextSub = undefined;
  }
  if (__DEV__ && dep.subsHead === link) {
    dep.subsHead = nextSub;
  }

  if (dep.subs === link) {
    dep.subs = prevSub;

    if (!prevSub && dep.computed) {
      dep.computed.flags &= ~EffectFlags.TRACKING;
      for (let l = dep.computed.deps; l; l = l.nextDep) {
        removeSub(l, true);
      }
    }
  }

  if (!soft && !--dep.sc && dep.map) {
    dep.map.delete(dep.key);
  }
}
```

### NOTIFIED

현재의 업데이트 주기(batch) 동안 Effect가 이미 실행 대기열에 추가되었는지를 나타낸다.

한 번의 데이터 변경으로 인해 여러 의존성이 동시에 트리거될 때, 동일한 이펙트가 여러 번 실행 대기열에 추가되는 것을 방지하는 중복 제거용 플래그이다. `batch()` 함수에서 설정되며, `endBatch()`에서 대기열을 처리한 후 해제된다.

```typescript
export function batch(sub: Subscriber, isComputed = false): void {
  sub.flags |= EffectFlags.NOTIFIED;
  if (isComputed) {
    sub.next = batchedComputed;
    batchedComputed = sub;
    return;
  }
  sub.next = batchedSub;
  batchedSub = sub;
}

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

### DIRTY

Effect의 의존성이 변경되어 값을 재계산해야하는 상태인지를 나타낸다.

`computed` 속성에서 핵심적인 역할을 한다. 의존하는 반응형 데이터가 변경되면 `computed`는 DIRTY 상태가 된다. `computed`의 값을 읽으려고 할 때 `isDirty()` 함수가 이 플래그를 확인하여, `DIRTY` 상태일 경우에만 값을 재계산한다. 재계산 후 이 플래그는 해제된다.

```typescript
function isDirty(sub: Subscriber): boolean {
  for (let link = sub.deps; link; link = link.nextDep) {
    if (
      link.dep.version !== link.version ||
      (link.dep.computed &&
        (refreshComputed(link.dep.computed) ||
          link.dep.version !== link.version))
    ) {
      return true;
    }
  }
  if (sub._dirty) {
    return true;
  }
  return false;
}

export function refreshComputed(computed: ComputedRefImpl): undefined {
  if (
    computed.flags & EffectFlags.TRACKING &&
    !(computed.flags & EffectFlags.DIRTY)
  ) {
    return
  }
  computed.flags &= ~EffectFlags.DIRTY

```
