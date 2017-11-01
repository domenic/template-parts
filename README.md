# `<template>` parts

This repository is a design/incubation repository for expanding the `<template>` element, allowing the creation of templates with specific "parts" that can be filled in later. It is derived from the discussion in [whatwg/html#2254](https://github.com/whatwg/html/issues/2254), and as it firms up, may eventually become a full-on proposal for the HTML Standard.

Major credit to the participants in that thread for originating many of these ideas.

## Overall goals

- We hope to standardize on a syntax for templating in HTML, similar to how JavaScript has standardized on template string syntax for templating JavaScript strings. We propose `{{foo}}` as the syntax for now. In general, we strive to emulate the flexibility of the JavaScript tagged template string model.
- We hope to let each framework and library define how the expressions inside the "template parts" `{{foo}}` are interpreted. The job of this feature is to hand off those expressions to libraries.

Problems that frameworks and template libraries encounter:

1. Loading, finding and/or associating templates with components
2. Finding expressions within attribute and text nodes
3. Parsing expressions
4. Evaluating expressions
5. Stamping templates into nodes
6. Re-evaluating templates to incrementally update previously stamped nodes.
7. Implementing control flow constructs like if and repeat
8. Implementing template composition or inheritance

This proposal hopes to address (2) and (5), while providing the building blocks for libraries to solve (3), (4), (7), and (8). (6) could go either way. (1) is out of scope.

## Template parts API

Given a template such as

```html
<template id="foo">
  <div class="foo {{y}}">{{x}} world</div>
</template>
```

we say that `{{x}}` and `{{y}}` correspond to **template parts**. They are exposed as JavaScript objects which can be manipulated in various ways. So far in this example we've identified two types of template parts: those in child node position, and those in attribute value position. Their respective APIs are:

* Both
  - `expression`: returns the expression string between the `{{` and `}}`
* Child node position
  - `replaceWith()`: replaces the part placeholder with the given strings and/or nodes, similar to [the existing `replaceWith()` method on nodes](https://dom.spec.whatwg.org/#dom-childnode-replacewith)
  - `parentNode`: the parent node in which this part lives
* Attribute value position
  - `value`: getter/setter for the value to replace the part placeholder
  - `attribute`: the `Attr` in which it lives

Ryosuke proposes adding `value` to the child node position template part as well, as an alias for `replaceWith(string)`. That may be a good idea for ease of use, but I have left it out for now, as `replaceWith()` feels like the more "core" API for child node position template parts.

We could add many more possible APIs, especially for the child node position template part, which could borrow from `Node`, the [`NonDocumentTypeChildNode` mixin](https://dom.spec.whatwg.org/#nondocumenttypechildnode), or the [`ChildNode` mixin](https://dom.spec.whatwg.org/#childnode). For now we stick with the above minimal set.

### Example code

Assume we somehow (see below) get `x` and `y` objects representing the template parts corresponding to `{{x}}` and `{{y}}`. Then:

```js
// Equivalent to div.textContent = "hello world"
// (or maybe it keeps "hello" and " world" as separate text nodes)
x.replaceWith("hello");

// Inserts an empty span and also text node in place of the {{x}} part
x.replaceWith(document.createElement("span"), "hello");

x.expression === "x";
x.parentNode === div;

// Equivalent to div.setAttribute("foo bar")
y.value = "bar";

y.expression === "y";
y.attribute === div.getAttributeNode("class");
```

### More types of template parts

The above is a very small list of the possible kinds of template parts. At the other extreme, we could go as far as to have a separate kind of part for every [tokenization state](https://html.spec.whatwg.org/#tokenization) of the HTML parser. (This is a bad idea.)

Probably we should proceed by defining an MVP of reasonable template part kinds, and some hard and fast rules for when they are recognized. Other instances of the `{{ }}` syntax would be left alone.

That is, we _don't_ need to support every possibility. When someone tries to do `<{{tagName}} class="foo">`, we just say that isn't supported, and [treat this as the HTML parser would normally](http://software.hixie.ch/utilities/js/live-dom-viewer/?saved=5415) instead of creating a special "tag name template part".

We definitely also need a whole-attribute template part, to be able to handle boolean attributes.

A possible other candidate would be a separate whole-attribute template part, for cases like `foo={{bar}}`, as opposed to attribute value position `foo="{{bar}}"`. Would this actually need to be separate, one wonders? It seems many frameworks make the distinction between these two syntaxes, so we can try to draw inspiration from them to gather use cases.

## `<template>` API

### The `template.instantiate()` method

A new method, `HTMLTemplateElement.prototype.instantiate(processor, params)` is created. Here is an example of this method in action, with a focus on the `processor` argument:

```js
function processor(parts, params) {
  for (const part of parts) {
    if (part.attribute) {
      part.value = params[part.expression];
    } else {
      part.replaceWith(params[part.expression]);
    }
  }
}

const result = document.querySelector("#foo").instantiate(processor, {
  x: "Hello",
  y: "bar"
});
```

The intended result here is that `result` is a document fragment (sorta; see below) equivalent to

```html
<template id="foo">
  <div class="foo bar">Hello world</div>
</template>
```

This processor function does a very simple substitution, looking up the "expression" as a key in the provided object literal. A more complex one could support e.g. dotted property access expressions like `{{x.prop}}`.

We could even have an overload that takes one argument and assumes a processor function similar to this one:

```js
const result = document.querySelector("#foo").instantiate({
  x: "world",
  y: "bar"
});
```

Then this feature could be used out of the box with no framework or library support.

### Updating instantiated templates

The `instantiate()` method above returns a `TemplateInstance`, which is a `DocumentFragment` subclass with some additional abilities.

Internally, it stores a list of the parts that came with the original template from which it was created.

Externally, it exposes a method `update(processor, params)` which will use the stored knowledge of the original template parts to update the template, replacing the previously-given param values with the results of applying the processor again. So for the above example, calling

```js
result.update(processor, { x: "Goodbye", y: "baz" });
```

would mutate the `TemplateInstance` to be a document fragment equivalent to

```html
<template id="foo">
  <div class="foo baz">Goodbye world</div>
</template>
```

This can then be done repeatedly.

**Open question: since document fragments, and thus presumably `TemplateInstance`s, disappear when inserted into an actual DOM tree, does this actually work?** It seems like it wouldn't help most applications, which need to update the page's DOM tree, not just in-memory document fragments.

Justin instead proposes making it easy to introspect the `<template>` element's parts, and then using existing incremental-DOM libraries to update. [He gives a code example](https://github.com/whatwg/html/issues/2254#issuecomment-272308819) based on adding `hasExpression` and `expression` properties to `Text`, `Element`, and `Attr`; I'm not sure if this could be adapted to use the parts API somehow.

## Nested templates/declarative templating?

It would be ideal if there were some way to allow instantiation to be configured by an attribute in the `<template>` markup. Such as:

```html
<template proccessor="fancy-template">
  <ul>
    <template proccessor="for-each" items="{{items}}">
      <li class={{class}} data-value={{value}}>{{label}}</li>
    </template>
  </ul>
</template>
```

It would be ideal if calling `outerTemplate.instantiate(params)` would invoke some proccessor by the name `fancy-template`, and also some specialized proccessor for the name `for-each` on the inner template. This would allow libraries to associate proccessors declaratively, including ones for common nested-template use cases like loops or conditionals.

(Why are loops and conditions natural as nested `<template>`s? Because they are semantically very template-like: they are chunks of DOM that may or may not be stamped out one or more times. Thus, it's natural to take advantage of the machinery we are proposing here to make them work.)

We could make this work in a variety of ways:

- Customized built-in elements that derive from `HTMLTemplateElement` (using `is=""` instead of `proccessor=""`), which can install their own default processor through a new hook
- Some sort of second global registry, apart from the custom elements registry: e.g. `HTMLTemplateElement.proccessors.define("fancy-template", processor)`
- Using the actual global object as a global registry, so that `processor="fancy-template"` looks for and invokes `window["fancy-template"]`
- Just relying on framework code to handle this, so that they figure out how to properly translate `proccessor=""` attributes into the appropriate first argument to pass to `instantiate()`

The trickiest part here is figuring out how the nested templates interact, e.g. what order they are processed in, and how they are represented in the processor functions. One proposal was to have a new type of template part that represents the nested template, with its own API.

## Parser problems

Even with the `<template>` element, HTML has some terrible parsing rules that hurt us. Consider

```html
<template>
  <table>
    {{theadGoesHere}}
    <tbody><!-- ... --></tbody>
  </table>
</template>
```

Ideally one would be able to use the above framework to insert a `<thead>` element at the part indicated by `{{theadGoesHere}}`. But, the parser gets in our way. [It produces a DOM tree like the following](http://software.hixie.ch/utilities/js/live-dom-viewer/?saved=5418):

```
TEMPLATE
  #text: {{theadGoesHere}}
  TABLE
    TBODY
      #comment: ...
    #text:
  #text:
```

If we want to allow this kind of substitution, then it seems we can't parse template parts as if they are normal text nodes which are then treated specially by `template.instantiate()`. We'd need to treat them specially at the level of the parser, which is historically quite hard to change.

Alternately, we could say that a case like this is not supported. Are there major use cases this hurts, besides tables?

A terrible hack is to change the syntax from `{{x}}` to `<!--x-->`, since comment nodes already parse in the way we want. This is so ugly though...
