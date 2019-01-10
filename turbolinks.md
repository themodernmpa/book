## Turbolinks

You might have heard of Turbolinks before, it's included as a gem by default with Ruby on Rails applications. Turbolinks isn't a framework, it's a javascript library that you layer on top of any existing web site that uses a few techniques to speed up page rendering, and more importantly, it gives you a *persistent javascript process* without you having to pay the SPA tax.

### How it Works

Turbolinks operates by adding event listeners for `a` tag click events. When a user clicks a link, instead of the browser issuing a regular GET request and navigating the user to the new location, the Turbolinks click handler instead makes an AJAX request for the same resource. Turbolinks then takes the returned DOM response, swaps the new `body` into the current page, updates the current URL, and merges the contents of the `head` element. Boom, new page is displayed, but your browser didn't refresh.

This gives us few key capabilities:

1. The new page renders a bit faster. The browser does not need to unload and re-parse any CSS or Javascript, and because no _actual_ navigation took place. Common layout elements The browser appears to "magically" update just a bit faster. This is why the package is called Turbo*links*.
2. Any Javascript state you may have stored on the first page will persist to the 2nd page. You now have a persistent JS runtime, page navigation will not stop any internal timers, hydrated state, WebRTC phone calls, etc. This is the part we really care about.

This technique is not without its downsides. Hijacking links removes a browsers' native feedback mechanisms, so you'll have to do some extra work to tell your users "something his happening right now". Adding Turbolinks to an existing site with a fair amount of JS functionality can also result in memory leakage, as you are probably relying on page loads to remove event listeners and flush stored state. We'll go over some techniques to mitigate these issues.

### A Persistant JS runtime

The biggest benefit Turbolinks gives you is the ability to rely on your JS runtime continuing between page loads. Say you're building an online music site clone. You want to let users play music tracks, but also browse playlists, view artist information, and in general use your web app while listening to music. You can build out all the CRUD pretty easily, but keeping an audio player going while navigating around is pretty tricky without resorting to frames.

This is a point where you typically start thinking about a SPA, but maybe all you really need is the ability to start an audio stream, and keep that audio stream in sync with a set of UI controls as your users navigate through page loads.

Turbolinks gives you the ability to build the CRUD portions of your app quickly and easily with the traditional server-side techniques you're already capable of doing, while also enabling you to build more complex interactions on the frontend for the isolated components that require that rich interactivity.

The Turbolinks documentation is very thorough, give it a read (we won't repeat all that good information here). What we would like to present here are some lessons we have learned over the years working with the library.

### Turbolinks is Backend Agnostic

While Turbolinks ships by default in Rails, it is not Rails-specific. You can add Turbolinks to any web application that responds with full page DOM, and it will perform well right out of the box. There's no reason you can't use the library on top of your Laravel, Django, or Struts website. Handling 302 redirects takes a little extra work, but it's not too much.

### Working with non-GET requests

Turbolinks automatically attaches itself to all `a` tags, handling GET requests out of the box. But if you want to make a form POST, a pseudo DELETE request, or anything non GET-related, you'll need to handle that yourself. We'll go into this more in the chapter about Working with Remote Responses TODO: Write

### Turbolinks Caching & Page Previews

TODO: This section is pretty heavyweight and probably fits better on its own? It's pretty close to talking about how to work with a Turbolinks + Stimulus component lifecycle.

One of the ways in which Turbolinks adds to the responsiveness of your web application is via an internal history store. The URL and DOM of up to 10 pages are cached in this store, and if Turbolinks detects that your user is navigating to a previously viewed page, via back button or page link, it will inject the cached copy back into the page and then perform the actual page load.

This can lead to some weird behavior if you're not writing your code in anticipation this fact. You could see weird behavior like:

- Flashes of outdated content. A counter incremented due to a user action is decremented when navigating back, then re-incremented again when the true page load happens.
- Flashes of manipulated content. It's a weird transition to hit the back button and see your old content displayed, then have the remote page fetched and see a loading spinner, then finally see your old content again.
- Double-binding of JS behavior. When a cached page is restored to the DOM it will still trigger any bound Mutation Observers, or other execution tied to the Turbolinks lifecycle. It's easy to get into a situation where a module binds to the DOM, manipulates content, that content gets cached then restored, and now the same module re-binds on top of its own manipulated content, and chokes on the modified DOM.

To mitigate these problems, we've developed a few simple rules to follow when handling cached pages.

1. Put the DOM back how you found it. Turbolinks fires events prior to caching a pages DOM. Subscribe to that event in your module, and unbind, dismount, and otherwise clean up after yourself before Turbolinks snapshots the page.
2. Avoid binding behavior when Turbolinks is displaying a cached page. The `body` element is annotated with a `data-turbolinks-preview` data attribute when the preview is being displayed. If you have a series of modules that load remote content on display, it might be best to not trigger that behavior when a cached page is being displayed.
3. Style previews differently. You can scope your module CSS with `body[data-turbolinks-preview]` to enable you to do a few tricks like hiding action buttons when the preview is shown (such as buttons that prompt a Modal to appear), and greying-out certain areas of the page to indicate "this is a preview". 
4. Don't cache pages that won't benefit from caching. Turbolinks allows you to opt out of caching for individual pages with the simple inclusion of a `meta` tag. If you don't think there's a benefit to showing a stale copy of a page, don't let Turbolinks cache it! Is there really helpful to allow a potentially out of date shopping cart checkout page get stored in a browser-side cache? Maybe it's best if that page is always fetched fresh from the server.

### Loading Indicators

A good rule of thumb is, if a user interaction will prompt an async action, plan on displaying an indicator. 

Turbolinks comes with a loading bar built-in for page navigation. By default it appears after 500ms of waiting for a page load and persists until the new DOM is ready to insert. We prefer to shorten the gap to around the 100ms range. Even if your server-side interactions are blazing fast, we would much rather display a loading bar that disappears quickly than have the user wonder "is it working?" and start stabbing at links in an attempt to get the content they want.

For any self-loading functionality (like a widget that updates itself with current weather information), spinners are a good solution. A little CSS works magic. You don't need anything complex, but you should have _something_ telling the user that content is coming.

If you have a button that triggers an async load that _isn't_ a page navigation, a good solution is to disable the button and replace the text with "Loading...". For bonus points you can animate the ellipses. 

The lack of native browser feedback during page loads isn't just a Turbolinks issue, it's a problem with any JS-based web interaction. We encourage you to find a solution and apply it consistently across your app.

### Asset changes & forced reloads

Once you've got a persistent frontend, you'll have to think about how you deploy changes and ensuring sure your users have the latest code. Turbolinks allows you to annotate script and css link tags with a `data-turbolinks-track` attribute. If you annotate your application asset bundles with these attributes, Turbolinks will initiate a true page reload when it detects a change in the version of those assets.

If you don't apply these attributes to your critical app asset files, you'll probably see errors due to double binding of JS, and double application of CSS, as Turbolinks will happily take the *new* asset tags and merge them into your existing `head` element.

We've also seen emergency bug fix deployments not squelch all errors as buggy javascript continues to happily run after the hotfix deployment, even if the users experiencing such errors are happily using the app. With full-page reloads being the only way to fetch new application code, having Turbolinks monitor for changes is essential.