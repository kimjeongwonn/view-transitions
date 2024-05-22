# 소개

부드러운 페이지 전환은 사용자가 페이지 A에서 페이지 B로 탐색할 때 [문맥을 유지](https://www.smashingmagazine.com/2013/10/smart-transitions-in-user-experience-design/)하는 데 도움이 되어 인지적 부담을 줄일 수 있으며, 로딩 시 [지연 현상을 줄이는](https://wp-rocket.me/blog/perceived-performance-need-optimize/#:~:text=1.%20Use%20activity%20and%20progress%20indicators) 느낌을 줄 수 있습니다.

https://user-images.githubusercontent.com/93594/184085118-65b33a92-272a-49f4-b3d6-50a8b8313567.mp4

# 왜 새로운 API가 필요한가?

일반적으로 웹의 탐색은 하나의 문서가 다른 문서로 전환되는 방식으로 이루어집니다. 브라우저는 [중간의 흰색 번쩍임을 제거하려고](https://developer.chrome.com/blog/paint-holding/) 노력하지만, 뷰 간의 전환은 여전히 갑작스럽고 급작스럽습니다. View Transitions 기능이 등장하기 전까지는, 개발자가 이 문제를 해결하려면 SPA 모델로 전환하지 않으면 안 되었습니다. 이 기능은 각 문서의 수명을 겹치지 않으면서도 두 문서 간에 애니메이션 전환을 만들 수 있는 방법을 제공합니다.

SPA로 전환하면 [CSS 전환](https://developer.mozilla.org/docs/Web/CSS/CSS_Transitions/Using_CSS_transitions), [CSS 애니메이션](https://developer.mozilla.org/docs/Web/CSS/CSS_Animations/Using_CSS_animations) 및 [웹 애니메이션 API](https://developer.mozilla.org/docs/Web/API/Web_Animations_API/Using_the_Web_Animations_API) 등 기존 기술을 사용하여 전환을 만들 수 있지만, 이는 생각보다 어렵기 때문에 대부분의 개발자와 프레임워크는 이를 회피하거나 제한된 방식으로만 사용합니다.

가장 간단한 전환 중 하나인 상태 간의 콘텐츠 블록 크로스페이드를 예로 들어봅시다.

이 작동을 위해서는 문서에 구식 콘텐츠와 새 콘텐츠가 동시에 존재하는 단계가 필요합니다. 구식 및 새 콘텐츠는 보통 서로 겹치는 위치에 있어야 하며, 페이지의 나머지 레이아웃과의 일관성을 유지해야 하기 때문에 이를 관리하기 위한 래퍼가 필요할 수 있습니다. 또 다른 이유는 둘 사이의 교차 페이드가 [mix-blend-mode: plus-lighter](https://jakearchibald.com/2021/dom-cross-fade/)를 사용하여 올바르게 작동하도록 하기 위함입니다. 그 다음, 구식 콘텐츠는 `opacity: 1`에서 `opacity: 0`으로 페이드 아웃하고, 새 콘텐츠는 `opacity: 0`에서 `opacity: 1`로 페이드 인합니다. 전환이 완료되면 구식 콘텐츠가 제거되고, 전환에 사용된 일부 래퍼도 제거할 수 있습니다.

그러나 이러한 간단한 예제에서도 여러 접근성과 사용성 문제점이 발생합니다. 구식 콘텐츠와 신식 콘텐츠가 동시에 존재하는 단계는 보조 기술 사용자들에게 혼란을 줄 수 있습니다. 전환은 시각적 유인이며, 스크린 리더와 같은 도구에는 보이지 않아야 합니다. 또한, 개발자가 이를 막지 못한 경우 사용자가 구식 콘텐츠와 상호작용할 가능성도 있습니다(예: 버튼 클릭). 전환 후 구식 콘텐츠가 제거되는 두 번째 DOM 변경은 DOM 변형으로 인해 동일 콘텐츠에 대한 추가 aria-live 발표를 발생시킬 수 있어 접근성 문제를 만들 수 있습니다. 특히, 구식 콘텐츠 DOM이 전환에 사용된 DOM과 동일하지 않을 경우 포커스 상태가 혼란스러워질 수 있습니다(가상 DOM을 다르게 인식할 경우, 특히 컨테이너가 변경된 경우).

콘텐츠가 크다면, 예를 들어 주요 콘텐츠라면, 개발자는 두 상태 간의 루트 스크롤 위치 차이를 처리해야 합니다. 최소한 하나의 콘텐츠는 전환이 완료될 때까지 스크롤 차이를 상쇄하기 위해 오프셋될 필요가 있습니다.

그리고 이건 단순한 크로스페이드일 뿐입니다. 페이지 구성 요소가 상태 간에 위치를 전환해야 할 때 상황은 훨씬 복잡해집니다. 사람들은 이 문제의 작은 부분을 해결하기 위해 [크고 복잡한 플러그인](https://greensock.com/docs/v3/Plugins/Flip/)을 만들고, 이보다 더 큰 라이브러리 위에 구축합니다. 그럼에도 불구하고, `overflow: hidden` 또는 유사한 방법으로 부모에 의해 클리핑되는 경우를 처리하지 못합니다. 이를 극복하기 위해 개발자는 애니메이션 요소를 `<body>`로 옮겨 자유롭게 애니메이션을 만들도록 합니다. 이를 위해 CSS를 변경하여 요소가 최종 위치의 DOM에 있는 것처럼 보이게 해야 합니다. 이는 개발자가 카스케이드를 사용하는 것을 단념하게 하고, 컨테이너 쿼리 같은 컨텍스트 스타일링 기능과 잘 맞지 않습니다.

사이트가 SPA라면 이것이 불가능한 것은 아니지만, _정말 어렵습니다_. 일반적인 탐색(일명 다중 페이지 애플리케이션 또는 MPA)에서는 불가능합니다.

View Transitions 기능은 개발자가 페이지 상태를 원자적으로 업데이트하면서도(예: DOM 변경 또는 문서 간 탐색을 통해) 두 상태 간의 고도로 맞춤화된 전환을 정의할 수 있게 해줍니다.

# MPA vs SPA 솔루션

[현재 사양](https://drafts.csswg.org/css-view-element-transitions-1/)과 Chrome에 구현된 내용은 SPA 전환에 중점을 두고 있습니다. 그러나 모델은 문서 간 탐색에도 작동하도록 설계되었습니다. 문서 간 탐색에 대한 구체적인 내용은 [여기](cross-doc-explainer.md)를 참고하세요.

이는 우리가 MPA 솔루션을 덜 중요하게 생각한다는 뜻이 아닙니다. 사실, [개발자들은 이것이 더 중요하다고 명확히 했습니다](https://twitter.com/jaffathecake/status/1405573749911560196). 우리는 프로토타입 제작의 용이성 때문에 SPA에 중점을 두었지만, 전체 모델은 약간의 차이가 있는 API를 사용하여 MPA에도 적용될 수 있도록 설계되었습니다.

# 크로스페이드 예제 재방문

앞서 설명한 것처럼, 기존 플랫폼 기능을 사용하여 크로스페이드 전환을 만드는 것은 생각보다 어렵습니다. View Transitions를 사용하여 이를 수행하는 방법은 다음과 같습니다:

```js
function spaNavigate(data) {
  // 이 API를 지원하지 않는 브라우저를 위한 폴백:
  if (!document.startViewTransition) {
    updateTheDOMSomehow(data);
    return;
  }

  document.startViewTransition(() => updateTheDOMSomehow(data));
}
```

_(API는 다음 섹션에서 자세히 설명됩니다)_

그리고 이제 상태 간의 크로스페이드가 가능합니다:

https://user-images.githubusercontent.com/93594/185887048-53f3695a-104d-4f50-86cc-95ce61422678.mp4

물론, 크로스페이드는 인상적이지 않습니다. 다행히도 전환을 사용자 정의할 수 있습니다. 그러나 그 전에 기본적인 크로스페이드가 작동한 방법을 알아봅시다:

# 크로스페이드가 작동한 방법

위의 코드 샘플을 보면:

```js
document.startViewTransition(() => updateTheDOMSomehow(data));
```

`document.startViewTransition()`이 호출되면, API는 페이지의 현재 상태를 캡처합니다. 여기에는 스크린샷 촬영이 포함되며, 이는 이벤트 루프의 렌더 단계에서 비동기로 진행됩니다.

완료되면, `document.startViewTransition()`에 전달된 콜백이 호출됩니다. 이때 개발자는 DOM을 변경합니다.

이는 렌더링이 일시 중지된 상태에서 진행되므로, 사용자는 새로운 콘텐츠의 플래시를 보지 않습니다. 다만, 렌더 중지는 가혹한 타임아웃을 가집니다.

DOM이 변경되면, API는 페이지의 새 상태를 캡처하고 다음과 같은 가상 요소 트리를 구성합니다:

```
::view-transition
└─ ::view-transition-group(root)
   └─ ::view-transition-image-pair(root)
      ├─ ::view-transition-old(root)
      └─ ::view-transition-new(root)
```

_(각 트리의 역할과 기본 스타일은 [문서의 뒷부분](#full-default-styles--animation)에 설명되어 있습니다)_

`::view-transition`은 페이지의 모든 다른 요소 위에 놓인 최상위 레이어에 위치합니다.

`::view-transition-old(root)`는 구 상태의 스크린샷이며, `::view-transition-new(root)`는 새로운 상태의 라이브 표현입니다. 둘 다 CSS 대체 콘텐츠로 렌더링됩니다.

구 이미지가 `opacity: 1`에서 `opacity: 0`으로, 새 이미지는 `opacity: 0`에서 `opacity: 1`로 애니메이션되면서 크로스페이드를 생성합니다.

애니메이션이 완료되면 `::view-transition`이 제거되어 최종 상태가 드러납니다.

백그라운드에서는 DOM이 이미 변경되었기 때문에, 구 콘텐츠와 새 콘텐츠가 동시에 존재하지 않는 상태가 되어 접근성, 사용성 및 레이아웃 문제를 피할 수 있습니다.

애니메이션은 CSS 애니메이션을 사용하여 수행되므로 CSS로 사용자 정의할 수 있습니다.

# 간단한 사용자 정의

위의 모든 가상 요소는 CSS로 타겟팅할 수 있으며, 애니메이션은 CSS로 정의되어 있으므로 기존 CSS 애니메이션 속성을 사용하여 수정할 수 있습니다. 예를 들어:

```css
::view-transition-old(root),
::view-transition-new(root) {
  animation-duration: 5s;
}
```

이 한 줄의 변경으로 페이드는 이제 매우 느려집니다:

https://user-images.githubusercontent.com/93594/185892070-f061181f-4534-46bd-99bb-657e2bce6cb9.mp4

또는 더 실용적으로는, [Material Design의 공유 축 전환](https://material.io/design/motion/the-motion-system.html#shared-axis)을 구현할 수 있습니다:

<!-- prettier-ignore -->
```css
@keyframes fade-in {
  from { opacity: 0; }
}

@keyframes fade-out {
  to { opacity: 0; }
}

@keyframes slide-from-right {
  from { transform: translateX(30px); }
}

@keyframes slide-to-left {
  to { transform: translateX(-30px); }
}

::view-transition-old(root) {
  animation: 90ms cubic-bezier(0.4, 0, 1, 1) both fade-out,
    300ms cubic-bezier(0.4, 0, 0.2, 1) both slide-to-left;
}

::view-transition-new(root) {
  animation: 210ms cubic-bezier(0, 0, 0.2, 1) 90ms both fade-in,
    300ms cubic-bezier(0.4, 0, 0.2, 1) both slide-from-right;
}
```

그리고 결과는 다음과 같습니다:

https://user-images.githubusercontent.com/93594/185893122-1f84ba5f-2c9d-46c6-9275-a278633f2e72.mp4

참고: 이 예에서는 애니메이션이 항상 오른쪽에서 왼쪽으로 이동하는데, 이는 뒤로 버튼을 클릭할 때 자연스럽지 않습니다. 탐색 방향에 따라 애니메이션을 변경하는 방법은 [문서의 뒷부분](#customizing-the-transition-based-on-the-type-of-navigation)에 설명되어 있습니다.

# 여러 요소 전환

이전 데모에서는 전체 페이지가 공유 축 전환에 참여합니다. 하지만 머리글이 전환될 때 슬라이드 아웃되고 다시 슬라이드 인되는 것이 옳지 않아 보입니다.

이를 해결하기 위해 View Transitions는 페이지의 일부를 독립적으로 애니메이션하도록 추출할 수 있도록, `view-transition-name`을 할당할 수 있습니다:

```css
.header {
  view-transition-name: header;
  contain: layout;
}
.header-text {
  view-transition-name: header-text;
  contain: layout;
}
```

_독립적으로 전환되는 요소는 `layout` 또는 `paint` 포함을 유지하고, 분할을 피해 단일 단위로 캡처될 수 있어야 합니다._

페이지는 이제 세 부분으로 캡처됩니다: 머리글, 머리글 텍스트 및 나머지 페이지('루트'로 알려짐).

https://user-images.githubusercontent.com/93594/184097864-40b9c860-480a-45ff-9787-62cebe68a078.mp4

이는 다음과 같은 전환을 위한 가상 요소 트리를 결과로 합니다:

```
::view-transition
├─ ::view-transition-group(root)
│  └─ ::view-transition-image-pair(root)
│     ├─ ::view-transition-old(root)
│     └─ ::view-transition-new(root)
│
├─ ::view-transition-group(header)
│  └─ ::view-transition-image-pair(header)
│     ├─ ::view-transition-old(header)
│     └─ ::view-transition-new(header)
│
└─ ::view-transition-group(header-text)
   └─ ::view-transition-image-pair(header-text)
      ├─ ::view-transition-old(header-text)
      └─ ::view-transition-new(header-text)
```

새로운 가상 요소는 처음과 동일한 패턴을 따르지만, 페이지의 하위 집합에 적용됩니다. 예를 들어, `::view-transition-old(header-text)`는 머리글 텍스트의 '스크린샷'이고, `::view-transition-new(header-text)`는 새로운 머리글 텍스트의 라이브 표현입니다. 이 경우, 머리글 텍스트 이미지가 동일하지만, 요소의 위치가 변경되었습니다.

추가적인 사용자 정의 없이 결과는 다음과 같습니다:

https://user-images.githubusercontent.com/93594/185895421-0131951f-c67b-4afc-97f8-44aa16cfbed7.mp4

탑 헤더가 정적임을 확인할 수 있습니다.

구 이미지와 신 이미지 간의 크로스페이드 외에도 다른 기본 애니메이션이 `::view-transition-group`을 이전 위치에서 새로운 위치로 변형시키면서 상태 간 너비와 높이를 전환합니다. 이는 머리글 텍스트가 상태 간 위치를 이동하도록 합니다. 다시 말해, 개발자는 이를 원하는 대로 CSS로 사용자 정의할 수 있습니다.

각 가상 요소의 고수준 목적:

-  `::view-transition-group` - 두 상태 간의 크기와 위치를 애니메이션합니다.
-  `::view-transition-image-pair` - 두 이미지가 올바르게 크로스페이드될 수 있도록 블렌딩 격리를 제공합니다.
-  `::view-transition-old` 및 `::view-transition-new` - 크로스페이드 시각 상태를 제공합니다.

가상 요소의 전체 기본 스타일과 애니메이션은 [문서의 뒷부분](#full-default-styles--animation)에 설명되어 있습니다.

# 전환 요소는 동일한 DOM 요소일 필요 없음

이전 예에서는 `view-transition-name`이 머리글과 머리글 텍스트에 대해 별도의 전환 요소를 만들기 위해 사용되었습니다. 이러한 요소들은 개념적으로는 동일한 요소이지만, 실제로는 DOM 변경 전후에 다를 수 있습니다.

예를 들어, 주요 비디오 임베드를 `view-transition-name`으로 지정할 수 있습니다:

```css
.full-embed {
  view-transition-name: full-embed;
  contain: layout;
}
```

그런 다음, 썸네일을 클릭할 때 동일한 `view-transition-name`을 전환 동안만 지정할 수 있습니다:

```js
thumbnail.onclick = () => {
  thumbnail.style.viewTransitionName = "full-embed";

  document.startViewTransition(() => {
    thumbnail.style.viewTransitionName = "";
    updateTheDOMSomehow();
  });
};
```

결과는 다음과 같습니다:

https://user-images.githubusercontent.com/93594/185897197-62e23bef-c198-4cd6-978e-c2e74892154b.mp4

이제 썸네일이 주요 이미지로 전환됩니다. 실제로 다른 요소이지만, 전환 API는 동일한 `view-transition-name`을 가진 요소로 취급합니다.

이는 하나의 요소가 다른 요소로 '변환'되는 경우에 유용하지만, 가상 DOM 비교에서 불일치로 인해 실제로는 변경되지 않은 경우에도 유용합니다.

또한, 이 모델은 상태 변경 시 모든 요소가 다른 DOM 요소가 되는 MPA 탐색에 필수적입니다.

# 전환 요소는 두 상태 모두에 존재할 필요 없음

일부 전환 요소는 한쪽 DOM 변경에만 존재할 수 있습니다. 예를 들어, 사이드바는 구 페이지에는 없지만 새 페이지에는 존재할 수 있습니다.

예를 들어, 요소가 '이후' 상태에서만 존재하는 경우 `::view-transition-old`가 없고, 기본적으로 `::view-transition-group`이 시작 위치에 없습니다.

# 탐색 유형에 따른 전환 사용자 정의

일부 경우, 캡처된 요소와 결과 애니메이션은 출발 페이지와 목표 페이지에 따라 다르며, 탐색 방향에 따라 달라져야 합니다.

https://user-images.githubusercontent.com/93594/184085118-65b33a92-272a-49f4-b3d6-50a8b8313567.mp4

이 예에서는 썸네일 페이지와 비디오 페이지 간의 전환이 비디오 페이지 간의 전환과 상당히 다르며, 뒤로 탐색할 때는 애니메이션 방향이 반전됩니다.

이를 처리하는 특정 기능은 없습니다. 개발자는 문서 요소에 클래스 이름을 추가하여 `view-transition-name`을 가진 요소와 사용할 애니메이션을 변경할 수 있습니다.

특히 [Navigation API](https://github.com/WICG/navigation-api)는 뒤로 탐색과 앞으로 탐색을 쉽게 구분할 수 있게 합니다.

# 애니메이션을 자바스크립트로 수행하기

`document.startViewTransition()`에서 반환된 `ViewTransition`의 `ready` 프로미스는 두 상태가 캡처되고 가상 요소 트리가 성공적으로 빌드되었을 때 이행됩니다. 이는 개발자가 [웹 애니메이션 API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Animations_API)를 사용하여 이러한 가상 요소들을 애니메이션할 수 있는 시점을 제공합니다.

예를 들어, 클릭한 지점에서 원형 공개 애니메이션을 만들려는 경우:

```js
let lastClick;
addEventListener("click", (event) => (lastClick = event));

async function spaNavigate(data) {
  // 이 API를 지원하지 않는 브라우저를 위한 폴백:
  if (!document.startViewTransition) {
    updateTheDOMSomehow(data);
    return;
  }

  const transition = document.startViewTransition(() => {
    // 클릭 위치를 가져오거나, 스크린 중앙을 기본값으로 사용합니다
    const x = lastClick?.clientX ?? innerWidth / 2;
    const y = lastClick?.clientY ?? innerHeight / 2;
    // 가장 먼 모서리까지의 거리를 계산합니다
    const endRadius = Math.sqrt(
      Math.max(x, innerWidth - x) ** 2 + Math.max(y, innerHeight - y) ** 2
    );

    updateTheDOMSomehow(data);
  });

  animateTransition(transition);

  // spaNavigate는 DOM 업데이트 시에 해제되어야 합니다,
  // 전환이 완료될 때가 아니라.
  return transition.updateCallbackDone;
}

async function animateTransition(transition) {
  await transition.ready;

  document.documentElement.animate(
    {
      clipPath: [
        `circle(0 at ${x}px ${y}px)`,
        `circle(${endRadius}px at ${x}px ${y}px)`,
      ],
    },
    {
      duration: 500,
      easing: "ease-in",
      // 애니메이션할 가상 요소를 지정합니다
      pseudoElement: "::view-transition-new(root)",
    }
  );

  return transition.finished;
}
```

그리고 결과물입니다:

https://user-images.githubusercontent.com/93594/184120371-678f58b3-d1f9-465b-978f-ee5eab73d120.mp4

# 같은 출처 내 문서 간 전환

이 섹션은 [자체 설명서](cross-doc-explainer.md)로 이동했습니다.

# 기존 개발자 도구와의 호환성

이 기능은 가상 요소 및 CSS 애니메이션과 같은 기존 개념을 기반으로 구축되었기 때문에, 이 기능을 위한 도구도 기존 개발자 도구와 잘 맞아야 합니다.

Chrome의 실험적 구현에서는 기존 애니메이션 패널을 사용하여 전환을 디버그할 수 있으며, 요소 패널에서는 가상 요소가 노출됩니다.

https://user-images.githubusercontent.com/93594/184123157-f4b08032-3b4f-4ca3-8882-8bea0e944355.mp4

# 프레임워크와의 호환성

DOM 업데이트는 비동기적으로 수행될 수 있어야 합니다. 이는 프레임워크가 마이크로태스크 뒤에 상태 업데이트를 큐에 배치하는 경우를 대비하기 위한 것입니다. 이는 `document.startViewTransition()` 콜백에서 프로미스를 반환함으로써 쉽게 수행할 수 있습니다. 예를 들어, 비동기 함수를 사용하여 이를 쉽게 달성할 수 있습니다:

```js
document.startViewTransition(async () => {
  await updateTheDOMSomehow();
});
```

그러나 위의 패턴은 개발자가 DOM 업데이트를 담당하는 경우를 가정하고 있는데, 이는 대부분의 웹 프레임워크에서는 그렇지 않습니다. 이 API의 프레임워크 호환성을 평가하기 위해, [이 설명서에서 다루는 데모 사이트](https://http203-playlist.netlify.app/)는 Preact를 사용하여 제작되었으며, [리액트 스타일의 훅](https://github.com/jakearchibald/http203-playlist/blob/main/src/shared/utils.ts#L53)을 사용하여 이 API를 래핑하고, React/Preact에서 사용할 수 있도록 했습니다.

프레임워크가 DOM 업데이트 시점을 알리는 기능을 제공하는 한, 전환 API는 프레임워크와 함께 작동할 수 있습니다.

# 오류 처리

이 기능은 전환을 DOM 변경에 대한 향상 기능으로 구축되었습니다. 예를 들면:

```js
document.startViewTransition(async () => {
  await updateTheDOMSomehow();
});
```

API는 `document.startViewTransition()` 콜백이 호출되기 전에 전환할 수 없는 오류를 감지할 수 있습니다. 예를 들어, 동일한 `view-transition-name`을 가진 두 개의 요소가 발견되거나 하나의 전환 요소가 API와 호환되지 않는 방식으로 분할된 경우입니다. 이 경우, 전환을 생성할 수 없더라도 `document.startViewTransition()` 콜백은 여전히 호출됩니다. 이는 전환보다는 DOM 변경이 더 중요하기 때문이며, 전환을 생성할 수 없다고 해서 DOM 변경을 방해하는 것은 이유가 되지 않습니다.

그러나 전환을 생성할 수 없는 경우, 반환된 `ViewTransition`의 `ready` 프로미스가 거부됩니다.

오류 감지는 `document.startViewTransition()`이 콜백을 받는 이유이기도 합니다. 이는 개발자가 유사한 상황에서 `updateTheDOMSomehow()`가 오류를 발생시킬 경우를 방지하기 위해서입니다. 이는 전환 초기화와 오류 케이스를 쉽게 관리할 수 있게 합니다.

[Navigation API](https://wicg.github.io/navigation-api/#ref-for-dom-navigateevent-intercept%E2%91%A0%E2%91%A5)와 [Web Locks API](https://w3c.github.io/web-locks/#ref-for-dom-lockmanager-request-name-options-callback%E2%91%A0)도 같은 이유로 이 패턴을 사용합니다.

# 잉크 오버플로우 처리

요소는 `box-shadow`와 같은 이유로 인해 테두리 박스 밖으로 그림을 그릴 수 있습니다.

`::view-transition-old` 및 `::view-transition-new`은 원래 요소의 테두리 박스 크기이지만, 전체 잉크 오버플로우는 이미지에 포함됩니다. 이는 `object-view-box`를 통해 달성되어, 교체된 요소가 자신의 경계를 벗어나 그릴 수 있도록 합니다.

# `width` 및 `height` 애니메이션

`::view-transition-group`은 기본적으로 `width`와 `height`를 애니메이션하며, 이는 보통 메인 스레드에서 애니메이션이 실행됨을 의미합니다.

그러나 개발자 편의상 `width`와 `height`가 고의적으로 선택되었습니다. 이는 `object-fit` 및 `object-position`과 잘 맞아 떨어지기 때문입니다.

https://user-images.githubusercontent.com/93594/184117389-3696400b-b381-478b-9837-888650c6d217.mp4

이 예에서는 4:3 썸네일이 16:9 메인 이미지로 전환됩니다. 이는 `object-fit`을 사용하면 상대적으로 쉬운 일이지만, 변환만을 사용하여 수행하는 것은 복잡합니다.

이 가상 요소 트리의 단순한 특성 덕분에 이러한 애니메이션은 메인 스레드 밖에서 실행될 수 있습니다. 그러나 개발자가 레이아웃을 요구하는 것을 추가하면, 예를 들어 테두리 등을 추가하면 애니메이션은 메인 스레드로 되돌아갑니다.

# 전체 기본 스타일 및 애니메이션

## `::view-transition`

기본 스타일:

```css
::view-transition {
  // 이 요소를 "스냅샷 뷰포트"와 정렬시킵니다. 이는 모든 축소 가능한 UI(예: URL 바, 루트 스크롤바, 가상 키보드)가 숨겨질 때의 뷰포트입니다.
  position: fixed;
  top: -10px;
  left: -15px;
}
```

## `::view-transition-group(*)`

기본 스타일:

```css
::view-transition-group(*) {
  /*= 모든 인스턴스의 스타일 =*/
  position: absolute;
  top: 0px;
  left: 0px;
  will-change: transform;
  pointer-events: auto;

  /*= 인스턴스별로 생성된 스타일 =*/

  /* 새 요소의 치수 */
  width: 665px;
  height: 54px;

  /* 새 요소의 뷰포트 위치에 배치하는 변환. */
  transform: matrix(1, 0, 0, 1, 0, 0);

  writing-mode: horizontal-tb;
  animation: 0.25s ease 0s 1 normal both running page-transition-group-anim-main-header;
}
```

기본 애니메이션:

```css
@keyframes page-transition-group-anim-main-header {
  from {
    /* 오래된 요소의 치수 */
    width: 600px;
    height: 40px;

    /* 오래된 요소의 뷰포트 위치에 배치하는 변환. */
    transform: matrix(2, 0, 0, 2, 0, 0);
  }
}
```

## `::view-transition-image-pair(*)`

기본 스타일:

```css
::view-transition-image-pair(*) {
  /*= 모든 인스턴스의 스타일 =*/
  position: absolute;
  inset: 0px;

  /*= 인스턴스별로 생성된 스타일 =*/
  /* 구 이미지와 신 이미지가 있는 경우, 교차 페이드를 돕기 위해 설정됩니다. 이는 격리의 성능 비용이 있기 때문에 조건부로 이루어집니다. */
  isolation: isolate;
}
```

기본 애니메이션: 없음.

## `::view-transition-old(*)`

이것은 구 요소의 캡처 표시를 나타내는 교체 요소로, 구 요소의 자연 비율이 적용됩니다.

```css
::view-transition-old(*) {
  /*= 모든 인스턴스의 스타일 =*/
  position: absolute;
  inset-block-start: 0px;
  inline-size: 100%;
  block-size: auto;
  will-change: opacity;

  /*= 인스턴스별로 생성된 스타일 =*/

  /* 구 이미지와 신 이미지가 있는 경우, 교차 페이드를 돕기 위해 설정됩니다. 이는 격리의 성능 비용이 있기 때문에 조건부로 이루어집니다. */
  mix-blend-mode: plus-lighter;

  /* 요소의 레이아웃 크기의 이미지가 되도록 하지만, 이미지 데이터의 잉크 오버플로우와 언더플로우를 변경합니다. */
  object-view-box: inset(0);

  animation: 0.25s ease 0s 1 normal both running blink-page-transition-fade-out;
}
```

이 요소의 `block-size`는 자동으로 되어 있으며, 컨테이너가 높이를 변경함에 따라 이미지를 스트레칭하지 않습니다. 개발자는 원하는 경우 이를 변경할 수 있습니다.

기본 애니메이션:

```css
@keyframes page-transition-fade-out {
  from {
    opacity: 0;
  }
}
```

## `::view-transition-new(*)`

기본 애니메이션:

```css
@keyframes page-transition-fade-in {
  to {
    opacity: 0;
  }
}
```

# 향후 작업

이 기능에는 우리가 적극적으로 생각하고 있지만 완전히 설계되지 않은 부분이 있습니다.

## 중첩 전환 그룹

현재 설계에서는 각 `::view-transition-group`이 `::view-transition`의 자식입니다. 이는 대부분의 경우 정말 잘 작동합니다.

…(잠재적 예제 생략)…

대략적인 계획으로 중첩을 허용하기 위해 선택적 설정을 사용할 예정입니다:

```css
.container {
  view-transition-name: container;
  contain: paint;
}
.child-item {
  view-transition-name: child-item;
  contain: layout;
  page-transition-style-or-whatever: nested;
}
```

이를 통해 컨테이너가 형제로 되는 대신:

```
::view-transition
├─ …
├─ ::view-transition-group(container)
│  └─ ::view-transition-image-pair(container)
│     └─ …
└─ ::view-transition-group(child-item)
   └─ ::view-transition-image-pair(child-item)
      └─ …
```

…`child-item`은 전환 요소인 가장 가까운 부모에서 중첩됩니다:

```
::view-transition
├─ …
└─ ::view-transition-group(container)
   ├─ ::view-transition-image-pair(container)
   │  └─ …
   └─ ::view-transition-group(child-item)
      └─ ::view-transition-image-pair(child-item)
         └─ …
```

## 더 세밀한 스타일 캡처

기본적으로 요소는 이미지로 캡처됩니다. 이는 만약 둥근 상자가 다른 크기 상자로 전환된다면, 전환 중에 모서리가 약간 불완전하게 확대될 수 있음을 의미합니다.

이것은 실제로는 그렇게 나쁘지 않으며, 특히 빠른 전환에서는 문제가 되지 않습니다. 그리고 개발자는 `clip-path`와 같은 커스텀 애니메이션을 만들어 이를 해결할 수 있습니다. 그러나 우리는 이 문제를 해결하기 위해 컴퓨터 스타일을 캡처하는 다른 설정 모드를 고려하고 있습니다.

이 모드에서는 요소 안의 콘텐츠는 여전히 이미지로 남지만, 요소 자체는 `border-radius`와 `box-shadow`와 같은 스타일을 갖게 됩니다.

다만, 이러한 애니메이션은 레이아웃을 포함하기 때문에 메인 스레드에서 실행되어야 합니다.

## 더 나은 가상 요소 선택자

이 기능은 중첩된 가상 요소를 사용합니다. 이는 `::before::marker`와 같은 요소보다 더 많은 중첩 레벨을 가집니다.

현재는 모든 가상 요소가 루트 요소에서 접근 가능하여, 트리 구조를 제대로 표현하지 않습니다. 그러나 이를 완전히 표현하면 다음과 같은 선택자가 되지만:

```css
::view-transition-group(foo)::image-wrapper::old-image {
  /* … */
}
```

우리는 새로운 결합자 제안을 통해 하위 가상 요소를 쉽게 선택할 수 있도록 제안하고 있습니다 [링크](https://github.com/w3c/csswg-drafts/issues/7346).

```css
::view-transition-group(foo) :>> old-image {
  /* … */
}
```

이는 CSS 네스팅과 잘 맞아떨어집니다:

```css
::view-transition-group(foo) {
  & :>> old-image {
    /* … */
  }
  & :>> new-image {
    /* … */
  }
}
```

## 특정 요소에 대해 타겟팅된 전환

현재 설계에서는 전환이 문서 전체에 작동합니다. 그러나 개발자들은 이 시스템을 제한된 특정 요소 하나에만 사용하고자 하는 관심을 표현했습니다. 예를 들어, 두 개의 독립된 구성 요소가 전환을 수행하는 경우입니다.

이 문제는 현재 진행 중입니다 [링크](https://github.com/WICG/view-transitions/issues/52) 및 [대략적인 제안](https://github.com/WICG/view-transitions/blob/main/scoped-transitions.md)에서 논의 중입니다.

# 보안/프라이버시 고려사항

아래의 보안 고려사항은 같은 출처 내 전환에 대해 다룹니다.

-  스크립트는 이미지 내의 픽셀 내용을 읽을 수 없습니다. 이는 문서가 교차 출처 콘텐츠(예: iframes, CORS 리소스 등) 및 제한된 사용자 정보(예: 방문한 링크, 검사의 사전 사용 등)를 포함할 수 있기 때문에 필요합니다.
-  캡처된 전환 요소가 '계산된 스타일 + 콘텐츠 이미지'로 캡처되는 경우, 컨테이너에 명시된 모든 외부 리소스(예: 배경 이미지)는 새 페이지의 문맥에서 다시 페치됩니다.

교차 출처 전환은 아직 정의되지 않았지만, 강하게 제한될 가능성이 있습니다.

# 상호작용성과 접근성

-  페이지 전환은 순전히 시각적인 기능입니다. 상호작용 측면에서 전환 요소는 원래 요소와 관계없이 `div`처럼 작동합니다. 개발자가 직접 전환 요소에 대화형 기능을 추가하는(예: `tabindex` 속성 추가) 방식으로 이 의도를 깨트릴 수 있습니다. 그러나 이는 추천되지 않습니다.
-  페이지 전환 단계는 스크린 리더와 같은 보조 기술에 숨겨져 있습니다.
-  DOM 렌더링이 억제되는 동안, 입력 처리도 일시 중지됩니다. 이는 사용자에게 시각적으로 보여지는 상태가 DOM 상태와 일치하지 않는 경우가 생기기 때문에 필요합니다. 
