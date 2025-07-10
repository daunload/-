# 효율적인 DOM 조작

## 새로운 요소를 만드는 것보다는 숨기기/표시를 지향하자

javascript로 요소를 제거하고 생성하는 대신 요소를 숨기고 표시하여 DOM 변경을 최소화하는 것이 성능 향상에 도움이 된다.

```js
el.classList.add('show')el.style.display = 'block'
```

### `display: none` 방식의 장점

#### 낮은 렌더링 비용

- `display: none`으로 설정된 요소는 DOM 트리에 존재하지만, 렌더 트리에서는 제외된다. 그렇기에 브라우저는 이 요소를 화면에 그리지 않고, Layout 또는 Reflow 에서도 무시한다.
- `display: block`으로 요소를 다시 보이게 할 때에는 이미 존재하는 요소의 css 속성만 변경하므로, 비교적 가벼온 Reflow만 발생한다.

#### 상태 및 이벤트 리스너 유지

- `display: none` 처리된 요소는 메모리에서 사리지지 않는다. `<input>` 태그라면 사용자가 입력했던 값이 남아있고, `<video>` 태그라면 재생하던 위치가 유지된다.
- 요소에 `addEventListener`로 추가된 이벤트 리스너들이 그대로 유지된다. 따라서 다시 보이게 했을 때 이벤트 핸들러를 추가할 필요가 없다.

### 동적 노드 추가/제거 방식의 단점

- DOM 구조가 변경될 때마다 전체 페이지의 레이아웃을 다시 계산하는 Reflow와 Repaint를 유발한다.
- 노드가 DOM에서 제거되면 해당 노드와 관련된 모든 정보(사용자 입력 값 등)가 사라진다. 다시 보여주기 위해 새로운 요소를 생성해야한다.
- 노드가 제거될 때 이벤트 리스너도 제거되기에 다시 생성하고 DOM에 추가할 때마다 이벤트 리스너도 붙여줘야 한다.

### 동적 추가 방식이 유용한 경우

- 초기 로딩 속도 향상을 위해서 무거운 컴포넌트는 동적 생성이 유리할 수 있다.
- 수천 개의 아이템을 가진 무한 스크롤의 경우, 보이지 않는 요소를 DOM에 유지하는 것은 메모리 낭비이기에 화면에서 벗어난 노드는 제거하는 가상 스크롤 기법 사용 필요

## 노드 생성을 일괄처리하는 `createDocumentFragment`를 활용하자

`createDocumentFragment`를 활용하면 모든 요소를 개별적으로 생성하여 삽입하지 않고 한 번에 삽입하여 Reflow와 Repaint를 최소화할 수 있다.

```js
const fragment = document.createDocumentFragment();
for (let i = 0; i < 1000; i++) {
  const li = document.createElement("li");
  li.textContent = `Item ${i}`;
  fragment.appendChild(li);
}
document.getElementById("myList").appendChild(fragment);
```

https://developer.mozilla.org/ko/docs/Web/API/Document/createDocumentFragment

## 텍스트 조작 시 `textContent`를 사용하자

DOM 요소의 텍스트를 변경할 때 `innerHTML`아나 `innerText` 보단 `textContent`를 사용하는 것이 더 빠르고 안전하다.

`innerHTML`: HTML을 파싱해야 하므로 느리고, XSS 공격에 취약할 수 있다.

`innerText`: 렌더링된 화면의 텍스트를 고려하므로, 스타일 계산으로 인한 Reflow가 발생하여 성능 저하의 원인이 될 수 있다.

`textContent`: 단순히 노드의 텍스트 콘텐츠만을 다루므로 가장 빠르고 효율적이다.

---

원문: https://frontendmasters.com/blog/patterns-for-memory-efficient-dom-manipulation/
