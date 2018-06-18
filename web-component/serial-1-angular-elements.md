## How Angular Elements implement Custom Element under the hood.

We already have a lot of articles describe how to create a `Custom Element` with `Angular Elements`, in this articles we will explain how `Angular` implements `Custom Element`.

### Custom Element

You can find all details about `Custom Element` [here](https://developers.google.com/web/fundamentals/web-components/customelements), we will go through what is `Custom Element` very quickly via an example (we will use the Custom Element v1 spec).

#### What is Custom Element

With Custom Elements, web developers can create new HTML tags, beef-up existing HTML tags, or extend the components other developers have authored.

For example, we want to create a `app-hello` tag, with a `name` and a `disabled` attribute.

We will use them like below just like use normal `HTML Element`. We will not use `ShadowDom` which is not must have in `Custom Element`, it will be another topic and we will talk in another article.

```html
<app-hello name="Custom Elements" disabled>
```

We will implement this `app-hello Custom Element` like this.

```js
class AppHello extends HTMLElement {
  constructor() {
    super();
    var disabledAttr = this.disabled ? 'disabled' : '';
    this.input = document.createElement('INPUT');
    this.button = document.createElement('INPUT');
    this.button.setAttribute('type', 'button');
    this.appendChild(this.input);
    this.appendChild(this.button);
  }

  get disabled() {
    return this.hasAttribute('disabled');
  }

  set disabled(val) {
    if (val) {
      this.setAttribute('disabled', '');
    } else {
      this.removeAttribute('disabled');
    }
  }

  connectedCallback() {
    if (this.disabled) {
      this.input.setAttribute('disabled', '');
    }
    this.input.setAttribute('value', this.name);
    this.btnClicked = function() {
      this.disabled = !this.disabled;
    };
    this.button.addEventListener('click', this.btnClicked);
  }

  disconnectedCallback() {
    this.button.removeEventListener('click', this.btnClicked);
  }

  attributeChangedCallback(attrName, oldVal, newVal) {
    if (attrName === 'disabled') {
      if (this.disabled) {
        this.input.removeAttribute('disabled');
      } else {
        this.input.setAttribute('disabled', '');
      }
    }
  }
}
```

This is a pretty simple `Custom Element`.
1. `app-hello` wrapper a input with a button.
2. the input will display the `name` attribute of `app-hello`.
3. and the input's `disabled` attribute will connect with `app-hello`'s.
4. also when clicked the button, the `disabled` attribute will be toggled.

### Use pure Angular to do this.

We know we can leverage the `Angular Element` power to implement such kind of `Custom Element` easily, but we want to show how `Angular` do under the hood, so we will not use `Custom Element` APIs, we will just use normal `Angular` and Html5 native CustomElement APIs.

First we will have an `Angular Component`, this is very simple and it is just the normal `Angular Component`.

```ts
@Component({
  selector: 'app-hello',
  template: ``,
})
```