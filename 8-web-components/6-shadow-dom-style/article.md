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

Same as `:host`, but applied only if the shadow host or any of its ancestors in the outer document matches the `selector`.

E.g. `:host-context(.dark-theme)` matches only if there's `dark-theme` class on `<custom-dialog>` on anywhere above it:

```html
<body class="dark-theme">
  <!--
    :host-context(.dark-theme) applies to custom-dialogs inside .dark-theme
  -->
  <custom-dialog>...</custom-dialog>
</body>
```

To summarize, we can use `:host`-family of selectors to style the main element of the component, depending on the context. These styles (unless `!important`) can be overridden by the document.

## Styling slotted content

Now let's consider the situation with slots.

Slotted elements come from light DOM, so they use document styles. Local styles do not affect slotted content.

In the example below, slotted `<span>` is bold, as per document style, but does not take `background` from the local style:
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

The result is bold, but not red.

If we'd like to style slotted elements in our component, there are two choices.

First, we can style the `<slot>` itself and rely on CSS inheritance:

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

Here `<p>John Smith</p>` becomes bold, because CSS inheritance is in effect between the `<slot>` and its contents. But in CSS itself not all properties are inherited.

Another option is to use `::slotted(selector)` pseudo-class. It matches elements based on two conditions:

1. That's a slotted element, that comes from the light DOM. Slot name doesn't matter. Just any slotted element, but only the element itself, not its children.
2. The element matches the `selector`.

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
