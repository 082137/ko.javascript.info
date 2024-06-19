# Shadow DOM 스타일링

Shadow DOM에는 `<style>` 태그와 `<link rel="stylesheet" href="…">` 태그가 모두 포함될 수 있습니다. 후자의 경우, 스타일시트는 HTTP 캐시가 적용되어 동일한 템플릿을 사용하는 여러 구성 요소에 대해 다시 다운로드되지 않습니다.

일반적으로 로컬 스타일은 shadow 트리 내부에서만 작동하고, 문서 스타일은 그 외부에서 작동합니다. 하지만 몇 가지 예외가 있습니다.

## :host

`:host` 선택자는 shadow 호스트(Shadow 트리를 포함하는 요소)를 선택할 수 있게 해줍니다.

예를 들어, 중앙에 배치되어야 하는 `<custom-dialog>` 요소를 만든다고 가정해봅시다. 이를 위해 `<custom-dialog>` 요소 자체를 스타일링해야 합니다.

바로 그것이 `:host`가 하는 일입니다.

```html run autorun="no-epub" untrusted height=80
<template id="tmpl">
  <style>
    /* 스타일은 내부에서 custom-dialog 요소로 적용됩니다. */
    :host {
      position: fixed;
      left: 50%;
      top: 50%;
      transform: translate(-50%, -50%);
      display: inline-block;
      border: 1px solid red;
      padding: 10px;
    }
  </style>
  <slot></slot>
</template>

<script>
customElements.define('custom-dialog', class extends HTMLElement {
  connectedCallback() {
    this.attachShadow({mode: 'open'}).append(tmpl.content.cloneNode(true));
  }
});
</script>

<custom-dialog>
  안녕하세요!
</custom-dialog>
```

## Cascading

Shadow 호스트(`<custom-dialog>` 자체)는 light DOM에 존재하므로 문서 CSS 규칙의 영향을 받습니다.

만약 어떤 속성이 로컬에서 `:host`로 스타일링되고 문서에서도 스타일링되었다면, 문서 스타일이 우선권을 가집니다.

예를 들어, 문서에 다음과 같은 스타일이 있을 경우:
```html
<style>
custom-dialog {
  padding: 0;
}
</style>
```
...그렇다면 `<custom-dialog>`에는 패딩이 없게 됩니다.

이는 매우 편리한데, `:host` 규칙에서 "기본" 구성 요소 스타일을 설정하고, 문서에서 쉽게 이를 재정의할 수 있기 때문입니다.

예외는 로컬 속성이 `!important`로 지정된 경우입니다. 이러한 속성에 대해서는 로컬 스타일이 우선권을 가집니다.


## :host(selector)

 `:host`와 같지만, shadow 호스트가 `selector`와 일치할 때에만 적용됩니다.

예를 들어, `centered` 속성이 있는 경우에만 `<custom-dialog>`을 가운데 정렬하고 싶다면:

```html run autorun="no-epub" untrusted height=80
<template id="tmpl">
  <style>
*!*
    :host([centered]) {
*/!*
      position: fixed;
      left: 50%;
      top: 50%;
      transform: translate(-50%, -50%);
      border-color: blue;
    }

    :host {
      display: inline-block;
      border: 1px solid red;
      padding: 10px;
    }
  </style>
  <slot></slot>
</template>

<script>
customElements.define('custom-dialog', class extends HTMLElement {
  connectedCallback() {
    this.attachShadow({mode: 'open'}).append(tmpl.content.cloneNode(true));
  }
});
</script>


<custom-dialog centered>
  가운데 정렬된 경우입니다!
</custom-dialog>

<custom-dialog>
  가운데 정렬되지 않은 경우입니다!
</custom-dialog>
```

이제 추가된 가운데 정렬 스타일은 첫 번째 대화 상자인 `<custom-dialog centered>`에만 적용됩니다.

## :host-context(selector)

`:host`와 유사하지만, shadow 호스트나 그 외부 문서의 조상 요소 중 하나가 `selector`와 일치할 때에만 적용됩니다.

예를 들어, `:host-context(.dark-theme)`은 `<custom-dialog>` 또는 그 상위의 어느 곳이라도 `dark-theme` 클래스가 있는 경우에만 일치합니다:

```html
<body class="dark-theme">
  <!--
    :host-context(.dark-theme) applies to custom-dialogs inside .dark-theme
  -->
  <custom-dialog>...</custom-dialog>
</body>
```

요약하면,  `:host` 계열 선택자를 사용하여 컴포넌트의 주요 요소를 문맥에 따라 스타일링할 수 있습니다. 이러한 스타일들은 (`!important`로 지정하지 않는 한) 문서에서 재정의할 수 있습니다.

## Styling slotted content

이제 슬롯을 고려해 보겠습니다.

슬롯된 요소들은 light DOM에서 가져오기 때문에 문서 스타일을 사용합니다. 로컬 스타일은 슬롯된 콘텐츠에 영향을 주지 않습니다.

아래 예시에서는 슬롯된 `<span>`이 문서 스타일에 따라 굵게 표시되지만, 로컬 스타일에서 `배경색`은 적용되지 않습니다:
```html run autorun="no-epub" untrusted height=80
<style>
*!*
  span { font-weight: bold }
*/!*
</style>

<user-card>
  <div slot="username">*!*<span>John Smith</span>*/!*</div>
</user-card>

<script>
customElements.define('user-card', class extends HTMLElement {
  connectedCallback() {
    this.attachShadow({mode: 'open'});
    this.shadowRoot.innerHTML = `
      <style>
*!*
      span { background: red; }
*/!*
      </style>
      Name: <slot name="username"></slot>
    `;
  }
});
</script>
```

결과는 굵게 표시되지만 빨간색으로는 표시되지 않습니다.

컴포넌트 내의 슬롯된 요소를 스타일링하려면 두 가지 선택지가 있습니다.

첫째, `<slot>` 자체를 스타일링하고 CSS 상속을 의존할 수 있습니다:

```html run autorun="no-epub" untrusted height=80
<user-card>
  <div slot="username">*!*<span>John Smith</span>*/!*</div>
</user-card>

<script>
customElements.define('user-card', class extends HTMLElement {
  connectedCallback() {
    this.attachShadow({mode: 'open'});
    this.shadowRoot.innerHTML = `
      <style>
*!*
      slot[name="username"] { font-weight: bold; }
*/!*
      </style>
      Name: <slot name="username"></slot>
    `;
  }
});
</script>
```

여기서 `<p>John Smith</p>`는 굵게 표시됩니다. 이는 `<slot>`과 그 내용물 사이에 CSS 상속이 적용되기 때문입니다. 그러나 CSS 자체에서 모든 속성이 상속되는 것은 아닙니다.

또 다른 옵션은 `::slotted(selector)` 가상 클래스를 사용하는 것입니다. 이는 두 가지 조건을 기반으로 요소를 선택합니다:

1. 라이트 DOM에서 슬롯된 요소여야 합니다. 슬롯 이름은 중요하지 않습니다. 단순히 어떤 슬롯된 요소든 가능하지만, 해당 요소 자체에만 적용되며 그 자식 요소는 포함되지 않습니다.
2. `selector`와 일치해야 합니다.

In our example, `::slotted(div)` selects exactly `<div slot="username">`, but not its children:

```html run autorun="no-epub" untrusted height=80
<user-card>
  <div slot="username">
    <div>John Smith</div>
  </div>
</user-card>

<script>
customElements.define('user-card', class extends HTMLElement {
  connectedCallback() {
    this.attachShadow({mode: 'open'});
    this.shadowRoot.innerHTML = `
      <style>
*!*
      ::slotted(div) { border: 1px solid red; }
*/!*
      </style>
      Name: <slot name="username"></slot>
    `;
  }
});
</script>
```

Please note, `::slotted` selector can't descend any further into the slot. These selectors are invalid:

```css
::slotted(div span) {
  /* our slotted <div> does not match this */
}

::slotted(div) p {
  /* can't go inside light DOM */
}
```

Also, `::slotted` can only be used in CSS. We can't use it in `querySelector`.

## CSS hooks with custom properties

How do we style internal elements of a component from the main document?

Selectors like `:host` apply rules to `<custom-dialog>` element or `<user-card>`, but how to style shadow DOM elements inside them?

There's no selector that can directly affect shadow DOM styles from the document. But just as we expose methods to interact with our component, we can expose CSS variables (custom CSS properties) to style it.

**Custom CSS properties exist on all levels, both in light and shadow.**

For example, in shadow DOM we can use `--user-card-field-color` CSS variable to  style fields, and the outer document can set its value:

```html
<style>
  .field {
    color: var(--user-card-field-color, black);
    /* if --user-card-field-color is not defined, use black color */
  }
</style>
<div class="field">Name: <slot name="username"></slot></div>
<div class="field">Birthday: <slot name="birthday"></slot></div>
```

Then, we can declare this property in the outer document for `<user-card>`:

```css
user-card {
  --user-card-field-color: green;
}
```

Custom CSS properties pierce through shadow DOM, they are visible everywhere, so the inner `.field` rule will make use of it.

Here's the full example:

```html run autorun="no-epub" untrusted height=80
<style>
*!*
  user-card {
    --user-card-field-color: green;
  }
*/!*
</style>

<template id="tmpl">
  <style>
*!*
    .field {
      color: var(--user-card-field-color, black);
    }
*/!*
  </style>
  <div class="field">Name: <slot name="username"></slot></div>
  <div class="field">Birthday: <slot name="birthday"></slot></div>
</template>

<script>
customElements.define('user-card', class extends HTMLElement {
  connectedCallback() {
    this.attachShadow({mode: 'open'});
    this.shadowRoot.append(document.getElementById('tmpl').content.cloneNode(true));
  }
});
</script>

<user-card>
  <span slot="username">John Smith</span>
  <span slot="birthday">01.01.2001</span>
</user-card>
```



## Summary

Shadow DOM can include styles, such as `<style>` or `<link rel="stylesheet">`.

Local styles can affect:
- shadow tree,
- shadow host with `:host`-family pseudoclasses,
- slotted elements (coming from light DOM), `::slotted(selector)` allows to select  slotted elements themselves, but not their children.

Document styles can affect:
- shadow host (as it lives in the outer document)
- slotted elements and their contents (as that's also in the outer document)

When CSS properties conflict, normally document styles have precedence, unless the property is labelled as `!important`. Then local styles have precedence.

CSS custom properties pierce through shadow DOM. They are used as "hooks" to style the component:

1. The component uses a custom CSS property to style key elements, such as `var(--component-name-title, <default value>)`.
2. Component author publishes these properties for developers, they are same important as other public component methods.
3. When a developer wants to style a title, they assign `--component-name-title` CSS property for the shadow host or above.
4. Profit!
