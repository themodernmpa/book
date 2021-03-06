## Turbolinks

You might have heard of Turbolinks before, it's included as a gem by default with Ruby on Rails applications. Turbolinks isn't a framework, it's a javascript library that you layer on top of any existing web site that uses a few techniques to speed up page rendering, and more importantly, it gives you a *persistent javascript process* without you having to pay the SPA tax.

### How it Works

Turbolinks operates by adding event listeners for `a` tag click events. When a user clicks a link, instead of the browser issuing a regular GET request and navigating the user to the new location, the Turbolinks click handler instead makes an AJAX request for the same resource. Turbolinks then takes the returned DOM response, swaps the new `body` into the current page, updates the current URL, and merges the contents of the `head` element. Boom, new page is displayed, but your browser didn't refresh.

This gives us few key capabilities that we will leverage in building a rich multi-page application.

### Faster Page Loading

Turbolinks equipped sites will render new HTML faster. The browser does not need to unload the current page, or re-parse any CSS or Javascript. Common layout elements will remane in-place, while what changed will appear to magically update, just like a traditional SPA. 

While Turbolinks will make page navigation faster, it won't speed up any other Javascript execution. It's still entirely possible to write slow, badly structured JS that gives the user a bad experience. 

### A Persistant JS runtime

The biggest benefit Turbolinks gives you is the ability to rely on your JS runtime continuing between page loads. Say you're building an online music site clone. You want to let users play music tracks, but also browse playlists, view artist information, and in general use your web app while listening to music. You can build out all the CRUD pretty easily, but keeping an audio player going while navigating around is pretty tricky without resorting to frames.

This is a point where you typically start thinking about a SPA, but maybe all you really need is the ability to start an audio stream, and keep that audio stream in sync with a set of UI controls as your users navigate through page loads.

With Turbolinks your users will never naturally reload the page, giving you the ability to rely on your JS state persisting during the user session. Because all your page loads respond with regular HTML, this gives you the ability to build the CRUD portions of your app quickly and easily with the traditional server-side techniques while also enabling you to build more complex interactions on the frontend for the isolated components that require that rich interactivity.

### Turbolinks Events

As Turbolinks replaces the browsers' native load cycle, it provides a variety of events that you can hook into to replace & augment these native events. The two most commonly missed native events are the `ready` and `unload` events fired on `document`. If you've used jQuery you're used to seeing `$(document).on("ready")`, typically a lot of functionality is tied to this event. In a Turbolinks-enabled site this event will fire once on initial page load, and then never again.

You can view the full list of Turbolinks events in [their docs](https://github.com/turbolinks/turbolinks#full-list-of-events). For our purposes we're most concerned with replacing `ready` and `unload`.

`"turbolinks:load"` is your direct replacement for `document.ready`. This event is fired after the initial page load, and then after every page visit. If you have jQuery behaviors that need to initialize on a page load, bind to this event.

`"turbolinks:render"` is similar to `load`, but this event is fired any time the DOM is updated by Turbolinks. This includes cached page re-rendering, so be careful about binding too much functionality here, as cached pages are typically replaced very quickly with a fresh copy from the server.

`"turbolinks:before-cache"` is fired right before Turbolinks takes a snapshot of the current page and replaces it with new DOM. This event is your opportunity to cleanup any changes you've made.

`"turbolinks:click"` is an analog to `document.unload`. This event is triggered whenever the user takes an action that Turbolinks intercepts, before Turbolinks does anything. We have successfully used this event to do simple, but powerful things like annotating the body element with a new data attribute like `data-turbolinks-loading`, which in turn we use to style specific components when loading is taking place.

These events are critical for replacing the now defunct native events, but they become incredibly powerful when hooked directly into your Stimulus controllers, as we will discuss in [Chapter 7 - Application Structure](application_structure.md).

One word of caution: Executing slow, blocking Javascript inside any of these event handlers can significantly impact the user experience. Consider deferring the execution of any heavy lifts.

### Turbolinks Caching & Page Previews

One of the ways in which Turbolinks adds to the responsiveness of your web application is via an internal history store. The URL and DOM of up to 10 pages are cached in this store, and if Turbolinks detects that your user is navigating to a previously viewed page, via back button or page link, it will inject the cached copy back into the page prior to fetching fresh page content.

This behaviour is a significant change from the traditional full-page reload, or a native browser Back action, and it can lead to some weird behavior if you're not writing your code in anticipation this fact. You will see things like:

- Flashes of outdated content. A counter incremented due to a user action is decremented when navigating back, then re-incremented again when the true page load happens.
- Flashes of manipulated content. It's a weird transition to hit the back button and see your old content displayed, then have the remote page fetched and see a loading spinner, then finally see your old content again.
- Double-binding of JS behavior. When a cached page is restored to the DOM it will still trigger any bound Mutation Observers, or other execution tied to the Turbolinks lifecycle. It's easy to get into a situation where a module binds to the DOM, manipulates content, that content gets cached, and then on restore the same module re-binds on top of its own manipulated content, and chokes on the modified DOM.

To mitigate these problems, we've developed a few simple rules to follow when handling cached pages.

1. Stay tidy. Turbolinks fires the `turbolinks:before-cache` event prior to caching a pages DOM. Use this event to clean up anything you don't want to blink between the cache display & page load. 
2. Avoid binding behavior when Turbolinks is displaying a cached page. The `body` element is annotated with a `data-turbolinks-preview` attribute when the preview is being displayed, you can detect this and avoid executing functionality.
3. Style previews differently. You can scope your module CSS with `body[data-turbolinks-preview]` to enable you add styling to indicate that the page is a preview. Using CSS to hide actionable components can also be useful. In the situation where a link or button pops a modal, allowing the modal to be spawned from the preview for a long-loading page can result in a user spawning a modal that then gets automatically closed.
4. Don't cache pages that won't benefit from caching. Turbolinks allows you to opt out of caching for individual pages with the simple inclusion of a `meta` tag. If you don't think there's a benefit to showing a stale copy of a page, don't let Turbolinks cache it! Is there really helpful to allow a potentially out of date shopping cart checkout page get stored in a browser-side cache? Maybe it's best if that page is always fetched fresh from the server.

### Replacing Native Loading Indicators

Because Turbolinks hijacks all vanilla links, the browser will no longer give your users a loading indicator when they click a link. To mitigate this Turbolinks comes with a loading bar built-in for page navigation. By default it appears at the top of your page after 500ms of waiting, and persists until the new DOM is ready to insert. We prefer to shorten the delay to around the 100ms range. Even if your server-side interactions are blazing fast, we would much rather display a loading bar that disappears quickly than have the user wonder "is it working?" and start stabbing at links in an attempt to get the content they want.

For any self-loading functionality (ex: a widget that updates itself with current weather information), spinners are a good solution. A little CSS works magic. You don't need anything complex, but you should have _something_ telling the user that content is coming.

If you have a button that triggers an async load that _isn't_ a page navigation, a good solution is to disable the button and replace the text with "Loading...". For bonus points you can animate the ellipses. 

The lack of native browser feedback during page loads isn't just a Turbolinks issue, it's a problem with any JS-based web interaction. We encourage you to find a solution and apply it consistently across your app. A good rule of thumb is, if a user interaction will prompt an async action and you have no way of proactivly showing the success state, plan on displaying an indicator.

### Asset changes & forced reloads

Once you've got a persistent frontend, you'll have to think about how you deploy changes and ensure sure your users have the latest code. Turbolinks allows you to annotate script and css link tags with a `data-turbolinks-track` attribute. If you annotate your application asset bundles with these attributes, Turbolinks will initiate a true page reload when it detects a change in the version of those assets.

If you don't apply these attributes to your critical app asset files, you'll probably see errors due to double binding of JS, and double application of CSS, as Turbolinks will happily take the *new* asset tags and merge them into your existing `head` element.

We've also seen emergency bug fix deployments not squelch all errors as buggy javascript continues to happily run after the hotfix deployment, even if the users experiencing such errors are happily using the app. With full-page reloads being the only way to fetch new application code, having Turbolinks monitor for changes is essential.

### Managing Manual Navigation

It's not uncommon to want to perform some manual navigation via a Javascript component. The canonical example would be after handling a POST request asyncronously you usually want to send the user somewhere.

If you need to maintain your JS context, the usual usage of `window.location.assign(location)` or `window.location = location` isn't going to work for us as it bypasses Turbolinks entirely and performs a native reload.. You can however instruct Turbolinks to navigate to a desired location with the `Turbolinks.visit(location)` method, which will go through the same lifecycle as a link click. 

But `.visit()` isn't the only navigation method you're going to need. For the purposes of *testability* and *findability*, we like to encapsulate all navigation methods used in our app into a standalone module, like so:

```javascript
// ./helpers/navigate.js

export const navigate = {
  visit: newUrl => {
    Turbolinks.visit(newUrl)
  },
  nativeVisit: newUrl => {
    window.location.assign(newUrl)
  },
  reload: () => {
    Turbolinks.visit(window.location)
  },
  nativeReload: () => {
    window.location.assign(window.location)
  },
  updatePageUrl: newUrl => {
    Turbolinks.controller.pushHistoryWithLocationAndRestorationIdentifier(
      newUrl,
      Turbolinks.uuid()
    )
  }
}
```

`navigate.visit()` will perform a full Turbolinks page navigation event to the desired URL, preserving JS context. 

`navigate.nativeVisit()` provides access to the native navigation method, which will do a full page load.

`navigate.reload()` will do a Turbolinks visit to the current page. Fresh content will be fetched but the JS context will remain.

`navigate.nativeReload()` provides a native reload that will refresh teh current page and rebuild our JS context.

`navigate.updatePageUrl()` is a way to update the URL and Turbolinks page cache without navigating. Using this technique along with a tabbed-content or wizard-style interface will allow you to build rich on-page interfaces that honor the back and forward buttons.

**Testability**: When you write tests for your Javascript, asserting page navigation can be a sticky point. Using a module such as this makes the test easy, you can spy on the module and assert that the correct method is executed with the correct parameter.

**Findability**: Having your frontend JS littered with random `window.location = newUrl` and `Turbolinks.visit` can lead to some very frustrating debugging, and it's easy to let one-off usages slip through the cracks. With a standalone module you will always be able to look at whatever is importing `navigate` to know what could be issuing a page nav event.

### Isomorphic Rendering & SEO

One of the best parts about building a server-side rendered site with Turbolinks is the lack of extra effort for SEO. You don't need to worry about rendering your SPA server-side, isomorphic rendering concerns are out the door. If you're already writing semantic HTML and have an SEO strategy in place, you're done! Turbolinks is search engine friendly by default.

The fire-and-forget aspect of Turbolinks is a great boon to anyone used to writing a traditional multi-page application. You can layer the library on top of an already functioning application, make a few minor changes to how you attach your dynamic frontend logic, and for the most part you're done. You'll instantly gain a persistant javascript process for whatever tricky interactions you need to maintain, and you can see  immediate speed improvements. For 95% of your features you can forget that the library is even present, and for the remaining 10% it provides a simple, yet rich feature set with which you can write most anything.