## How Angular Elements implement Custom Element under the hood.

We already have a lot of articles describe how to create a `Custom Element` with `Angular Elements`, in this articles we will explain how `Angular` implements `Custom Element`.

### Custom Element

You can find all details about `Custom Element` [here](https://developers.google.com/web/fundamentals/web-components/customelements), we will go through what is `Custom Element` very quickly via an example (we will use the Custom Element v1 spec).

#### What is Custom Element

With Custom Elements, web developers can create new HTML tags, beef-up existing HTML tags, or extend the components other developers have authored.

For example, we want to create a `app-hello` tag, with a `name` attribute.

We will use them like below just like use normal `HTML Element`. We will not use `ShadowDom` which is not must have in `Custom Element`, it will be another topic and we will talk in another article.

```html
<app-hello name="Custom Elements">
```

We will implement this `app-hello Custom Element` like this by implementing `Custom Element callbacks`.

```js
class AppHello extends HTMLElement {
  constructor() {
    super();
  }

  // define which attributes need to be observed so attributeChangedCallback will be triggered
  // when according attribute changed.
  static get observedAttributes() {return ['name']; }

  // getter to do a attribute -> property reflection
  get name() {
    return this.getAttribute('name');
  }

  // setter to do a property -> attribute reflection
  set name(val) {
    this.setAttribute('name', val);
  }

  connectedCallback() {
    this.div = document.createElement('div');
    this.text = document.createTextNode(this.name || '');
    this.div.appendChild(this.text);
    this.appendChild(this.div);
  }

  disconnectedCallback() {
    this.removeChild(this.div);
  }

  attributeChangedCallback(attrName, oldVal, newVal) {
    if (attrName === 'name' && this.text) {
      this.text.textContent = newVal;
    }
  }
}

customElements.define('app-hello', AppHello);
```

This is a pretty simple `Custom Element`.
1. `app-hello` wrapper a text node.
2. the text node will display the `name` attribute of `app-hello`.

| callback | summary |
| --- | --- |
|constructor|initialize state or shadowRoot if needed, in this article, we don't need it|
|connectedCallback|Will be called when element is added to DOM (another case is upgrade, will not be discussed here), we will initialize our DOM and event listener here|
|disconnctedCallback|Will be called when element is removed from DOM, we will clear our DOM and event listener here|
|attributeChangedCallback|Will be called when attribute of the element change, we will update our internal dom element or state based on the changed attribute|

### Use pure Angular to do this.

We know we can leverage the `Angular Element` power to implement such kind of `Custom Element` easily, but we want to show how `Angular` do under the hood, so we will not use `Custom Element` APIs, we will just use normal `Angular` and Html5 native CustomElement APIs.

Here is what we need to do, we need to make `Angular` satisfy `Custom Element`'s requirement (callbacks).
So we have a `Angular HelloComponent`, we need to fill it into the `Custom Element` callbacks.

1. Here is a very simple normal `Angular Component`.

```ts
import { Component, Input } from '@angular/core';

@Component({
  selector: 'app-hello',
  template: `<div>{{name}}</div>`
})
export class HelloComponent  {
  @Input() name: string;
}

```

2. Then we will write a empty `Custom Element class`. Inside of each callback, we will use our `HelloComponent` to fulfill.
So basically we will create a `mapping` from `Angular Component` to `Custom Element`.
```js
class HelloComponentClass extends HTMLElement {
  constructor() {
    super();
  }

  static get observedAttributes() {
  }

  connectedCallback() {
  }

  disconnectedCallback() {
  }

  attributeChangedCallback(attrName, oldVal, newVal) {
  }
}
```

So the mapping looks like,

| callback | summary | angular part |
| --- | --- | --- |
|constructor|initialize internalstate|do some preparation work|
|connectedCallback|initialize View/Event Listener|Load Angular Component|
|disconnectedCallback|clear View/Event Listener|Destroy Angular Component|
|attributeChangedCallback|Handle attribute change|Handle @Input Change|

1. constructor

We need to initialize our `Angular Component` in `connectedCallback`, before that, we can put some preparation work here.
To initialize a `Angular Component` on the fly, does it sound famliar? Yes, it is the same as `dynamic component` mechanism.
About how to load `dynamic component`, you can find detail information [here](https://blog.angularindepth.com/here-is-what-you-need-to-know-about-dynamic-components-in-angular-ac1e96167f9e).

In constructor, we can do some preparation work for latter `initialization` in `connectedCallback`.

- initialize `componentFactory` based on the component definition.
- initialize `observerdAttributes` based on the component's inputs (@Input collection), so we can get attributeChangedCallback.

```ts
this.componentFactory = injector.get(ComponentFactoryResolver).resolveComponentFactory(component);
this.observedAttributes = componentFactory.inputs.map(input => input.templateName); // we use templateName to handle this case @Input('aliasName');
```

2. connectedCallback

- We will initialize our `Angular Component` here. This part is the same as `DynamicComponent`
- We will set the component's input initial values
- We will stream Component @Output to Element's EventEmitter, we will discuss this in serial2 `Angular Elements vs Custom Events`.
- Trigger change detection to render component
- Attach HostView to ApplicationRef

```ts
// first we need an componentInjector to initialize the component.
// here the injector is from outside of Custom Element, user can register some of their own
// providers in it.
const componentInjector = Injector.create({providers: [], parent: injector});
// analyze projectable nodes from element
// for example:
// <app-hello>
//   <div class="inside">some projection content</div>
// </app-hello>
const projectableNodes = injector.get(ComponentFactoryResolver).resolveComponentFactory(component);
// Here we got all we need, we will initialize the component
const componentRef = componentFactory.create(componentInjector, projectableNodes, element);

// Then we need to check whether we need to initialize value of component's input
// the case is, before Angular Element is loaded, user may already set element's property.
// those values will be kept in an initialInputValues map.
componentFactory.inputs.forEach(prop => componentRef[prop] = initialInputValues.get(prop));

// and we will also stream Component's @Output -> Element's EventEmitter
initializeOutputs();

// then we will trigger a change detection so the component will be rendered in next tick.
changeDetectorRef.detectChanges();

// finally we will attach this component's HostView to applicationRef
applicationRef.attachView(this.componentRef.hostView);
```

3. disconnectedCallback

Basically we will just `destroy` the componentRef here.

```ts
this.componentRef.destroy();
```

4.attributeChangedCallback

When attribute of element changed, we need to update Component's property accordingly, and trigger change detection.

```ts
attributeChangedCallback(attrName, oldVal, newVal) {
  if (this.componentRef[attrName] === newVal) {
    return;
  }
  this.componentRef[attrName] = newVal;
  this.changeDetectorRef.detectChanges();
}
```

That's all, we don't use `Angular Elements APIs`, but only use `Dynamic Components`, we implement a `Custom Element` in `Angular`.

So `@angular/elements` basically organize the logic above into several module and make them easy to read, to maintain, to extend.

- [create-custom-element.ts](https://github.com/angular/angular/blob/master/packages/elements/src/create-custom-element.ts), this module will implement `Custom Element callbacks`, and initialize a NgElementStrategy as the `bridge` to connect `Angular Component` to `Custom Elements`. Now we only have one `Strategy` which is `component-factory-strategy.ts`, which will use the logic we explained to do the bridge. In the future, we may have other `Strategy`, we can also implement the `Strategy` outselves.
- component-factory-strategy.ts, Creates and destroys a component ref using a component factory and handles change detection in response to input changes. The code we have explained above.

That's how `Angular Elements` implement `Custom Elements`, we will explain the `@Output` with `Custom Events` in the next article.

