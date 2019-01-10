# Stimulus

Stimulus is a small Javascript library that makes it easy to build interactive areas within your application without a lot of framework overhead.  The core building block of apps using Stimulus are Controllers which are bound to your HTML using data attributes.  This declarative style makes it easy to look at your HTML and clearly see what Stimulus components are interacting with it.

```html
<div data-controller="search">
  ...
</div>
```

 This snippit from the Stimulus web site shows you that the Search controller (search_controller.js by convention) is interacting with the DOM inside of this div element.  When combining this approach with your languages partials/includes etc it becomes quite trivial to build reusable components for your web app.

Stimulus uses a Mutation Observer to detect changes to the DOM and identify what controllers should be loaded at any time.  Only controllers that are referenced in the DOM are initialized, if you never render DOM containing a reference to a controller, that controller will never actually be instantiated.  MDN has a great page on Mutation Observers if you're looking for more information.

## Encapsulation

One of the great things about the Stimulus components is that they provide a simple and unopiniated mechanism for executing any Javascript that applies to specific areas of your DOM.  For example, Stimulus can be a light wrapper around a bunch of jQuery selectors and third party libraries like D3 which make it trivial for an engineer to build on working examples for fast development.  A benefit of encapsulating your Javascript in this manner is any differences in approach/methodology are isolated from each other to allow for the fastest iteration possible.  If it makes sense to use a one-off third party library in just one area, you can load it with Stimulus only when that DOM is loaded and interact with it in your own sandbox.  The benefit to your project is that you can use slightly different development techniques to create componenets, but you can also mix them together to create a cohesive whole.

While in the flexibility to do things differently within each controller is nice to have, Stimulus also provides conventions for accessing key DOM elements within each component, so you shouldn't have to fall back to jQuery for everything :). Use the separation but don't abuse it.

## Lifecycle

Stimulus controllers have a simple set of lifecycle methods: `initialize`, `connect` & `disconnect`.  Thats it - pretty simple!

`initialize`

Called when the component is first instantiated and not again unless the entire page is reloaded (which it generally won't be due to Turbolinks).

`connect`

Called when the controller is connected to DOM

`disconnect`

Called when the controller is disconnected from the DOM

Note that `connect` and `disconnect` may be called multiple times but `initialize` will only be called once unless the entire runtime is lost (due to a browser restart, page load etc.).

TODO: test above statement

To execute code in any one of these lifecycle events, you implement that method within your controller:

```javascript
import { Controller } from "stimulus"

export default class extends Controller {
  initialize() {
    // …
  }
  connect() {
    // …
  }
  disconnect() {
    // …
  }
}
```

## State Management

State management is one of the areas of front end development that needs more consideration from engineers.  Storing, maintaining and updating state is generally what the front end is all about!  It is possible to store state within your Stimulus controller, however managing insterstitial state (between user input and a "save" action) will be challenging.  If you need your state to persist between page loads, Stimulus uses `data` attributes:

```html
<div data-controller="user-profile"
     data-user-profile-number="8675309">
</div>
```

and in your controller:

```javascript
// controllers/user_profile_controller.js
import { Controller } from "stimulus"

export default class extends Controller {
  connect() {
    console.log(this.data.get("number"))
  }
}
```

Be aware that this can be quite verbose if you use multiple data attributes so keep it to the minimum.  While it may not be obvious, you can serialize a JSON object in your attribute as well, then in your controller you would deserialize it and in this way, pass multiple attributes:

```html
<div data-controller="user-profile"
     data-user-profile-user="{ 'number': 8675309, first_name: 'jenny' }">
</div>
```

and in your controller:

```javascript
// controllers/user_profile_controller.js
import { Controller } from "stimulus"

export default class extends Controller {
  connect() {
    this.user = JSON.parse(this.data.get("user"))
    console.log(this.user.number)
  }
}
```

Keep in mind the lifecycle of the component - whenever the DOM is loaded (either via page navigation or by an asynchronous request), the `connect` method of the controller is fired and the current version of the `user` object is loaded into the component instance.  This approach makes it trivial to simply throw away your controller once you change state on the server and replace it with new DOM containing the updated object which will rebind to the controller.

The simplicity of state management is maintained by not over-committing to your front end components. It is often simpler to re-render a partial with your DOM and let Stimulus rebind to your updated data attributes than to attempt to maintain a copy of state within your component that matches the expected state on the controller.  When it comes to state management, a single source of truth reduces complexity enormously.



- Component mount, dismount, teardown, pre-cache techniques for smooth page transitions

- Don't go outside components scope! Nonos regarding scope and area of domain.
- Storing State in the DOM vs in the Component
  - The DOM should always reflect what could come from teh server on a refresh
  - The component should track what happens after the refresh
- Singleton Components with data-turbolinks-permenant
- Warning - Too many operations in `connect()`