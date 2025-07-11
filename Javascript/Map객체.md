# Map 객체

키/값 쌍의 컬렉션으로, 키와 값 모두 임의의 ECMAScript 언어 값을 가질 수 있다. 각 키 값은 Map 컬렉션 내에 단 하나의 키/값 쌍으로 존재하며, 동일한 키 값의 구별에는 **SameValueZero 비교 알고리즘**이 사용된다.

Map은 해시 테이블 또는 컬렉션 요소 수에 대해 평균적으로 선형 이하의 접근 시간을 제공하는 메커니즘을 사용해 구현해야한다.

## SameValueZero 비교 알고리즘
Map/Set의 키 비교나 `Array.prototype.icludes` 등에서 쓰이는 "값 동일성" 알고리즘

```ts
// 이해를 위한 코드
function sameValueZero(x: any, y: any): boolean {
  if (typeof x !== typeof y) return false;
  if (typeof x === 'number') {
    return x === y || (Number.isNaN(x) && Number.isNaN(y));
  }
  // undefined, null, boolean, string, symbol, BigInt, object: SameValueNonNumber 로 비교
  return Object.is(x, y); // 단, Object.is 와 달리 +0/-0 분리는 무시됨
}
```

https://tc39.es/ecma262/multipage/abstract-operations.html#sec-samevaluezero

## 생성자로 호출되었을 때 동작과정
1. `new` 키워드가 없다면 `TypeError` 예외를 던진다.
2. `new` 키워드의 타겟과 Map.prototype을 프로토타입으로 갖는 새로운 빈 객체를 생성
   1. 이 객체에 내부에 Map의 엔트리를 저장할 `[[MapData]]` 슬롯을 준비해 둔다.
3. `[[MapData]]`슬롯에 빈 리스트 또는 빈 해시 테이블을 할당
4. 호출 시 생성자 파라미터(iterable)를 생략했거나 null을 넘겼다면 빈 Map을 즉시 반환
5. set 메서드가 함수가 아니라면 `TypeError` 예외를 던진다. (프로토타입 오몀 또는 서브클래싱 시 의도치 않은 동작을 방지하기 위한 안전장치)
6. 생성자 파라미터(iterable)가 존재한다면 순회하여 `map.set(key, value)`를 반복 호출한 이후에 최종 map을 반환한다.

## get 메서드 동작과정
1. this 가 Map 인스턴스가 아니라면 예외를 던진다.
2. key를 표준화한다 (`CanonicalizeKeyedCollectionKey`를 활용한다)
3. 내부 슬롯을 순회하며 `sameValue` 결과가 true인 레코드의 value를 반환
4. 모두 순회했지만 일치하는 키가 없다면 undefined 반환
> 순회라고 표현하지만 실제 구현은 해시 테이블로 구현된다.

## size 메서드 동작과정
1. this 가 Map 인스턴스가 아니라면 예외를 던진다.
2. key를 표준화한다 (`CanonicalizeKeyedCollectionKey`를 활용한다)
3. 내부 슬롯을 순회하며 비어 있지 않은 키를 하나씩 카운트한다.
4. 카운트한 값을 IEEE-754로 래핑한 값을 반환한다.
