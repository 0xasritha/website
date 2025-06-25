---
name: DNS Clobbering 
layout: post
collection: blog 
date: 2025-06-24
---

An XSS attack can usually be mitigated by enforcing Content Secuirty Policy and strict HTML sanitization (e.g. DOMPurify). Sites that allow users to inject HTML (e.g. blogging platforms, CMSs, etc.) often sanitize aggressively until all inputs seem safe. But the key word is “seem." 

An attacker can use benign HTML injection (no `<script>` tags, no inline event handlers, etc.) to bypass website logic and secuirty via <span style="color: #d8b862;"><b>DOM clobbering</b></span>. 😈


DOM clobbering abuses a browser feature called 
 <span style="color: #d8b862;"><b>named property access</b></span> (or 
  <span style="color: #d8b862;"><b>named properties</b></span> for short), where any element you given an `id` or `name` becomes a JavaScript property on objects like `window`, `document`, HTML forms etc.   

Let's do a simple example. Imagine a website contains the following code: 
```HTML 
<body>
    <form id="myForm" action="/submit" method="POST">
        <textarea id="userHTML"></textarea>
        <!--input id="submit"--> 
        <button id="render"> Render HTML </button>
    </form>  

    <script>
        const form = document.getElementById('myForm'); 
        const userHTML = document.getElementById('userHTML');
        const renderBtn = document.getElementById('render')

        renderBtn.addEventListener('click', e => { 
            e.preventDefault(); 

            const cleanHTML = DOMPurify.sanitize(userHTML.value);
            form.insertAdjacentHTML('beforeend', cleanHTML); // TODO: does this insert inside the form? 

            try { 
                form.submit(); 
            } catch (err) { 
                console.error("How did you get here?", err);
            }            
        }) 
    </script>
</body>
```

By injecting `<input name="submit">` within the form, an attacker can "clobber" - or replace - `form.submit()` reference from the default `submit` to the new `input` element. Given you cannot call `submit()` on the `<input>`, this will error out and log `"How did you get here?"`.

^#TODO: show example

DOM clobbering is very context dependent. To easily exploit it, you must first understand the nuances of **named property access** in the DOM API.  

## Named Properties per the [HTML](`https://html.spec.whatwg.org/multipage/dom.html#document`) + [Web IDL](`https://webidl.spec.whatwg.org/`) Living Standards

We will track the behavior of named properties in 4 different WebIDL interfaces (and their associated JS objects): `Window`, `Document`, `HTMLFormElement`, and `HTMLCollection`.

*NOTE: I know traversing the HTML + Web IDL specs is very tedius, so I included all relevant excerpts in the expandable blue boxes!*

### Prerequisite 

Feel free to skip this section if you already know the basics of Web IDL interfaces/spec?!

The Web IDL 

### Definitions 

A lot of the terminology you encounter in these specs is confusing.

Let's clarify some base definitions: 

#TODO: have code examples for these 
- **supported property names**: a set of strings (an object's proprety *keys*) that the interface promises to expose. This set is (#TODO: check) built once by traversing the interface object? in tree order and pulling out certain `name` or `id` attribute values
- **named elements**: the actual DOM `Element` node you will get back when you do a lookup for one of the names within the supported property names (ex. `window.bar` returns `<div id="bar">`). Depending on how many there are, you then return either a single element, an HTMLCollection, or (in the special <iframe> case) a WindowProxy (#TODO: check over).

- **named object**: the JS-level value you get back when 

So for each interface we visit, the browser will do the following: 
1. First, the browser builds/maintains the supported property names list for the Document interface.
2. Then, when you do document.foo, it gathers the named elements whose id or name is "foo".
3. Finally it applies the “named‑property lookup” rules (one element → return that element; multiple → HTMLCollection; none → undefined).

### Named Properties 

Let's say your website contains the following HTML element: `<img id="myImg">`. Running `window.myImg` in the developer console returns a reference to that specific `img` element. 

This is an example of a **named property** (or more formally, *"dynamic properties added on top of fixed properties to WebIDL platform objects"*). 

<details markdown="1" class="admonition note collapsible">
<summary markdown="span">
  **<a href="https://webidl.spec.whatwg.org/#idl-named-properties" 
          target="_blank" rel="noopener">§ 2.5.6.2 Named properties</a>** *(Web IDL Spec)*
</summary>

An interface that defines a named property getter is said to **support named properties**. By extension, a platform object is said to support named properties if it implements an interface that itself does.
   
If an interface supports named properties, then the interface definition must be accompanied by a description of the ordered set of names that can be used to index the object at any given time. These names are called the **supported property names**.
</details>

As per [§ 2.5.6.2 Named properties](`https://webidl.spec.whatwg.org/#idl-named-properties`), all interfaces (ex. `Window`, `Document`, etc.) that have support for named properties define two things:

1) a <span style="color: #d8b862;"><b>named property getter </b></span>  
2) a description of how the <span style="color: #d8b862;"><b>supported property names</b></span> is formed

Given each interface with support for named properties must define these two things, we can infer that not all WebIDL-defined interfaces that support them will behave the same.

Hence, we will need to reference the specs of our 4 interfaces indvidually. By the end of this section, we want to have the following table filled out: 

<table>
  <thead>
    <tr>
      <th>Interface</th>
      <th>Supported Property Names</th>
      <th>Lookup Algorithm</th>
      <th>Single vs. Collection</th>
      <th>Property Shadowing Behavior</th>
      <th>Case Sensitivity</th>
      <th>Return Value When Missing</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Window</td>
      <td></td>
      <td></td>
      <td></td>
      <td></td>
      <td></td>
      <td></td>
    </tr>
    <tr>
      <td>Document</td>
      <td></td>
      <td></td>
      <td></td>
      <td></td>
      <td></td>
      <td></td>
    </tr>
    <tr>
      <td>HTMLFormElement</td>
      <td></td>
      <td></td>
      <td></td>
      <td></td>
      <td></td>
      <td></td>
    </tr>
    <tr>
      <td>HTMLCollection</td>
      <td></td>
      <td></td>
      <td></td>
      <td></td>
      <td></td>
      <td></td>
    </tr>
  </tbody>
</table>



### Named properties of the `Window` interface 

Before we look at the behavior of named properties on `Window`, let's look at it's interface definition.

> [!TIP] 7.2.2 
> **HTML Spec: [§ 7.2.2 The `Window` Object](`https://html.spec.whatwg.org/multipage/nav-history-apis.html#the-window-object`)**  
>   
> 
> ```IDL 
> [Global=Window
> Exposed=Window, 
> LegacyUnenumerableNamedProperties]
> interface Window 
> ... 
> 
>  // Since this is the global object, the IDL named getter adds a NamedPropertiesObject exotic
>  // object on the prototype chain. Indeed, this does not make the global object an exotic object.
>  // Indexed access is taken care of by the WindowProxy exotic object.
>  getter object (DOMString name);
>
> ... 
> ```    
>  .#TODO: get rid of `.` for formatting   

The `Window` interface is annotated with the `Global` extended attribute. The comment starting with "Since this is a global object ..." implies that all `Global` interfaces have something called a `NamedPropertiesObject` exotic object added by the IDL named getter (#TODO: explain) in their prototype chain.

#### <u> `window`'s prototype chain </u>

> [!TIP]
> **Web IDL Spec: [§ 3.3.8. [Global]](`https://webidl.spec.whatwg.org/#Global`)**
>
> For these global interfaces, the structure of the prototype chain and how properties corresponding to interface members will be reflected on the prototype objects will be different from other interfaces. Specifically:
>
> 1. Any named properties will be exposed on an object in the prototype chain – the named properties object – rather than on the object itself. 
>
> 2. Interface members from the interface will correspond to properties on the object itself rather than on interface prototype objects. 
> 

Ok, so we know `Window` is a `Global` interface. Therefore, we can use `§ 3.3.8. Global`, to assume? that built in properties of `window` will be properties on the `window` object itself *(2)* (#TODO: is this citation correct?), whereas named properties will be part of a "named properties object" that lives within `window`'s prototype chain as a *special intermediate prototype object.* *(1)*

So, `window`'s prototype chain should follow this rough? structure to the Web IDL spec (#TODO: confirm): `window -> NamedPropertiesObject -> Object.proto -> null`. 

> [!NOTE]   
> If you go into the developer console right and type `window.__proto__` #TODO: put get prototype of ..?, you won't see our prototype chain we defined. This is because the Web IDL prototype chain is ... #TODO: finish 

---
Let's take a very brief detour. Every object lookup in Javascript (ex. `obj.foo`) will first check for `foo` as an "own property" (#TODO: define) on `obj`. If it finds one, that value is used and any other property named `foo` in `obj`'s prototype chain is ignored (this is called **prototype-chain shadowing**?). The code below shows an example: 

```Javascript 
// TODO: check over this code 
const proto = { foo: "from prototype" };
const obj   = Object.create(proto);

console.log(obj.foo);
// → "from prototype"  (no own property, so it falls back)

obj.foo = "own property";

console.log(obj.foo);
// → "own property"   (own property shadows the prototype one)
```
---
Understanding prototype-chain shadowing? is important because we can then trace `window`'s prototype chain to understand how named properties will get shadowed by global variables + builtins?

For example, lets say Website A has `var login = "Log in here";` and `<iframe name="login">` in it's code. Running `window.login` will return `"Log in here"`. (#TODO: verify. also insert example w/ built in if that is correct)

To further reinforce this behaviour, a green "Note" in  `§ 3.3.8. Global` states: 

>  Placing named properties o... #TODO: come back, idk if need


#### <u> The`NamedPropertiesObject` exotic </u> 
https://webidl.spec.whatwg.org/#named-properties-object

#TODO: come back idk if you need this 
#TODO: explain how the browser does not actually implement a NamedPropertiesObject? Also no idea if this is true ... 

#### <u> A named object </u>

But how do know *specifically* what should be placed in a `window`'s `NamedPropertiesObject`? 

We can call "the value" of a named property (what it references) a **named object** (named property --points to--> named object?). 

> [!NOTE]
> **HTML Spec: [§ 7.2.2.3 Named access on the Window object](`https://html.spec.whatwg.org/multipage/nav-history-apis.html#dom-window-nameditem-filter`)**
> 
> Named objects of `Window` object *window* with the name *name*, for the purposes of the above algorithm, consist of the following:
>
> - document-tree child navigables of window's associated `Document` whose target name is *name*;
>
> - `embed`, `form`, `img`, or `object` elements that have a name content attribute whose value is name and are in a document tree with window's associated `Document` as their root; and
> 
> - HTML elements that have an `id` content attribute whose value is *name* and are in a document tree with window's associated `Document` as their root.

To paraphrase  [§ 7.2.2.3 Named access on the Window object](`https://html.spec.whatwg.org/multipage/nav-history-apis.html#dom-window-nameditem-filter`), a website's? named objects can come from the following three places:   
1. child navigables (think `<iframe>`, `<frame>`, etc.) - we will formally define this shortly 
    - each child navigable has a browsing context with a target name - again this will be clear in just a sec 
2. `<embed>`, `<form>`, `<img>`, or `<object>` elements w/ a `name` attribute (ex. `<form name="foo">`)
3. any HTML element w/ an `id` attribute (etc. `<p id="bar">`)

So the references to these "named objects" will be returned when you run `window.NAME` or `window[NAME]`? 

#TODO: finish - lines 244-329 in Article-Draft 

There are two relevant algorithms the browser needs: 
1) Algorithm to create a `window`'s supported named properties (or the ordered set of named properties the object supports https://webidl.spec.whatwg.org/#dfn-supported-property-names)
2) Look up algorithm to find a nemed object when given a naemd property *window.name* 

#### Algorithm 1

> https://html.spec.whatwg.org/multipage/nav-history-apis.html#named-access-on-the-window-object #TODO: fix formatting 
> 
>  The Window object supports named properties. The supported property names of a Window object window at any moment consist of the following, in tree order according to the element that contributed them, ignoring later duplicates:
> 
> - window's document-tree child navigable target name property set;
> 
> - the value of the name content attribute for all embed, form, img, and object elements that have a non-empty name content attribute and are in a document tree with window's associated Document as their root; and
> 
> - the value of the id content attribute for all HTML elements that have a non-empty id content attribute and are in a document tree with window's associated Document as their root.

And the **document-tree child navigable target name property set** (a mouthful!) can be found in: 

```
https://html.spec.whatwg.org/multipage/nav-history-apis.html#named-access-on-the-window-object #TODO: fix formatting 

The document-tree child navigable target name property set of a Window object window is the return value of running these steps:

    Let children be the document-tree child navigables of window's associated Document.

    Let firstNamedChildren be an empty ordered set.

    For each navigable of children:

        Let name be navigable's target name.

        If name is the empty string, then continue.

        If firstNamedChildren contains a navigable whose target name is name, then continue.

        Append navigable to firstNamedChildren.

    Let names be an empty ordered set.

    For each navigable of firstNamedChildren:

        Let name be navigable's target name.

        If navigable's active document's origin is same origin with window's relevant settings object's origin, then append name to names.

    Return names.
```

Let's get some definitions out of the way: 
- **tree order** is just preorder, depth-first traversal of a tree (#TODO: confirm + link spec source). 
- **navigable container**: "represent something that can be navigated between documents" (https://html.spec.whatwg.org/multipage/document-sequences.html#infrastructure-for-sequences-of-documents). All navigable elements in HTML5? are: `iframe`, `frame`, `frameset` (#TODO: finish).
- therefore, **document-tree child navigables** of a a `document` are simply a list of `document`'s child navigable containers in tree order. 

For more clairty, let's do a rough implementation of Algorithm 1 in code: 

```javascript 

// get all navigable containers 

// get tree walker - make sure it returns in tree order  

const treewalker = "hello";  
```

#### Algorithm 2








---
```
FROM CHAT: confirm 
§ 3.7.4.1 [[GetOwnProperty]] says that before falling back to ordinary own‑ or prototype‑lookups, the object runs the “named property visibility algorithm.” If that says “yes, we have a named prop P,” it immediately returns that named property value.
```

- mention how being `[Global]` disables `[LegacyOverrideBuiltIns]` (+ cannot define named-property setters, indexed getters/setters or constructors??) -> is this relevant? 

### Named properties on the `Document` interface 

https://html.spec.whatwg.org/#the-document-object
- `LegacyOverrideBuiltIns`


To determine what named elements (#TODO: im not sure if we define this anywhere?) are available on `document`, we can read the following:

```
https://html.spec.whatwg.org/#dom-tree-accessors

Named elements with the name name, for the purposes of the above algorithm, are those that are either:

    Exposed embed, form, iframe, img, or exposed object elements that have a name content attribute whose value is name, or
    Exposed object elements that have an id content attribute whose value is name, or
    img elements that have an id content attribute whose value is name, and that have a non-empty name content attribute present also.

An embed or object element is said to be exposed if it has no exposed object ancestor, and, for object elements, is additionally either not showing its fallback content or has no object or embed descendants.
```

`<object>` tags in HTML are used to define a container for external resources (ex. web page, pictures, media players, etc.). So for example, `<object data="flower.jpg">` or `<object data="snippet.html">`. **Fallback content** within an `<object>` (and other elements such as `<embed>`) is defined as alternative content that is dispalyed whenever the primary content source or resource is not available.  
So based on that, let's define an **exposed object**: an `<embed>`/`<object>` element that is not currently displaying its fallback content, does not have any `<object>` + `<element>` descdendants (recurisve children? #TODO: confirm) + is not nested inside another exposed `<object>` or `<embed>`. 

To paraphrase, named elements in `document` are: 
1. exposed ...




To determine how this set/list? of **named elements** is formed, the spec states: 

```
The Document interface supports named properties. The supported property names of a Document object document at any moment consist of the following, in tree order according to the element that contributed them, ignoring later duplicates, and with values from id attributes coming before values from name attributes when the same element contributes both:

    the value of the name content attribute for all exposed embed, form, iframe, img, and exposed object elements that have a non-empty name content attribute and are in a document tree with document as their root;

    the value of the id content attribute for all exposed object elements that have a non-empty id content attribute and are in a document tree with document as their root; and

    the value of the id content attribute for all img elements that have both a non-empty id content attribute and a non-empty name content attribute, and are in a document tree with document as their root.
```


So, now we can translate the look up algorithm to find the value of a named property used by `document` into code:  

```
To determine the value of a named property name for a Document, the user agent must return the value obtained using the following steps:

    Let elements be the list of named elements with the name name that are in a document tree with the Document as their root.

    There will be at least one such element, since the algorithm would otherwise not have been invoked by Web IDL.

    If elements has only one element, and that element is an iframe element, and that iframe element's content navigable is not null, then return the active WindowProxy of the element's content navigable.

    Otherwise, if elements has only one element, return that element.

    Otherwise, return an HTMLCollection rooted at the Document node, whose filter matches only named elements with the name name.
```

``` JavaScript 
namedElements = Object.Array ...


// TODO: have examples where is not exposed object, and it is not added
// TODO: have an example HTMLCollection return 
```





### Named properties on the `HTMLFormElement` interface

The `HTMLFormElement` interface is annotated with the `LegacyOverrideBuiltIns` extended attribute

> **HTML Spec: [§ 4.10.3 The `form` element](`https://html.spec.whatwg.org/multipage/forms.html#the-form-element`)** 
>  
> ```IDL [Exposed=Window,
> LegacyOverrideBuiltIns,
> LegacyUnenumerableNamedProperties]
> interface HTMLFormElement : HTMLElement {
> ...
> ```
> .

#### <u> Legacy platform objects </u>

Remember, a **platform object** is just "an object implementing the [Web IDL] interface in question." So for the `HTMLFormElement` interface, the corresponding platform object is DOM objects that represent the `<form>` elements? 

> **Web IDL Spec: [§ 2.12. Objects implementing interfaces](`https://webidl.spec.whatwg.org/#idl-objects`)**  
> **Legacy platform objects** are platform objects that implement an interface which does not have a [Global] extended attribute, and which supports indexed properties, named properties, or both.

Given `HTMLFormElement` supports those properties and is not? `[Global]`, `<form>`? is a **legacy platform object**.

> **Web IDL Spec: [§ 3.4.7 [LegacyOverrideBuiltins]](`https://webidl.spec.whatwg.org/#LegacyOverrideBuiltIns`)**   
> If the `[LegacyOverrideBuiltIns]` extended attribute appears on an interface, it indicates that for a legacy platform object implementing the interface, properties corresponding to all of the object’s supported property names will appear to be on the object, regardless of what other properties exist on the object or its prototype chain. This means that named properties will always shadow any properties that would otherwise appear on the object. This is in contrast to the usual behavior, which is for named properties to be exposed only if there is no property with the same name on the object itself or somewhere on its prototype chain.
>

So for `HTMLFormElement` objects? and other `LegacyOverrideBuiltins` interfaces?, named properties shadow all other properties of the object (*including* built-ins, etc.). In fact, to reach the built-in when a colliding named property exists, you would have to do something like `Object.getPrototypeOf(obj).A.call(obj)` (#TODO: verify this). 

This logic seems quite fragile - but that's what happens when browsers must be backwards compatible. Older browsers exposed elements (ex. `<input name="foo">` on `form.foo`) no matter what else was present on the `form` object, and altering such behaviour would break a lot of already-present functionality on the web. (#TODO: verify all of this lol) 


#### Order of precedence (named property visibility algorithm)






### Recap 

## DOM Clobbering Techniques

Here is the simplest possible scenario of DOM clobbering: 

```JavaScript
var s = documment.createElement('script'); 
let config = window.globalConfig || (href: 'script.js'); 
s.src = config.href; 
document.body.appendChild(s);
// TODO: this code is identical to https://domclob.xyz/domc_wiki/ -> rewrite w/ similar premise, or credit

// TODO: insert place where use can insert HTML here but is completely sanitized (HTML encode ALL tags) - so the "vuln" immediately jumps out
```

After reading the above section, our vulnerability should immediately jump out. Although the attacker cannot an XSS payload, they can inject an HTML tag with `id="globalConfig"` (ex. `<a id="globalConfig" href="malicious.js">`), and as specified in our named property access?, the user will be able to control `s.src`. #TODO: explain better. 

#TODO: insert second scenario where named properties overshadow browser APIs (see second scenario in https://domclob.xyz/domc_wiki/)

