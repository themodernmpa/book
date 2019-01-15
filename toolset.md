## The Technologies & Techniques Involved

So what can you do without introducing Angular 17, or React+Redux+Redux-Thunk+React-Router+...? Maybe you've already built a few apps with jQuery and friends, and it works great up until the point when it becomes a hairball. Or maybe you've built SPA-based applications, and don't really want to embark on another journey managing two separate apps.

Our answer is, nearly everything. If you are able to take advantage of some of the new developments in JS-land, you can build yourself a very productive, but simple & familiar toolchain enabling you to keep your velocity up, and familarity high.

### To Webpack or not to Webpack

Your first major decision is how far into the NPM toolchain you want to go. We promote a *moderate* use of NPM & Friends. Webpack is pretty nice, albeit complex, but if you apply the same judicious rules you use in the rest of your tech stack, you can keep it sane and avoid most common issues.

- Add packages sparingly. `npm i` should be used for core tools only. The more you add here, the harder your life will get going forward.
- Don't get pollyfill happy. Use them sparingly, and only for what you really need. All of the new ES syntax is very nice, but depending on the browsers you have to support, you could end up inflating your final asset size by quite a bit.

Moving forward, we'll assume you're using an NPM-based package manager of some kind (we use Yarn).

### Prefer Serving HTML

We're going to build an app that prefers to serve HTML. We'll use a variety of techniques to manage how this HTML is handled, but your default assumption should always be, send HTML. This includes for most AJAX requests too! If you go too far down the rabbit hole of serving up JSON and parsing it on the front end you will slowly work your way into a position where you'd wish you were using Angular.

Your application server is very good at building HTML. Your web server is good at sending it. Your users' browser is amazingly good at parsing it. You should be sending HTML at every opportunity. If the standard is to send HTML, most of your transactions are reduced to a sequence of `send request, check response code, inject response into content`. This approach is simple to start with, and straightforward to maintain.

### Honor the URL

A fundamental part of building a modern multi-page application is to honor the URL in every way possible. If the user hits CTRL+R, they should reload *exactly as they are*. Do your best to always honor the URL, and to store any resumable state on the server so the page can be rebuilt server-side.

While modern SPA frameworks have gotten better about keeping the URL in sync 

#### Respond to DOM Presence

Trigger functionality based on the presence of elements in the DOM. Use a naming convention that is *not* tied to CSS classes (data attributes work great for this). Sniff the DOM, when you see an element, execute the appropraite callback to trigger functionality.

We'll use StimulusJS to do this automatically. It gives us a nice lifecycle for each component, but if you're doing this by hand you can achieve the same with a Mutation Observer, or if that is unavailable to you you can use a naming convention and callback registration scheme.

#### A Persistant JS Process

You've probably heard of Turbolinks before. It's a Basecamp tool, and has shipped as a part of Ruby on Rails for quite some time TODO: How long, and it has long been one of the first packages removed from Rails.

We're going to use it. Turbolinks will give you a persistant frontend JS process. You'll be able to persist timers and transient data across page loads with ease. With a few hand-rolled additions, you'll soon even forget that Turbolinks is around.

#### Skip the jQuery Soup

Stimulus is another Basecamp utility, new as of 2018, and it replaces a _lot_ of what people use jQuery for.

#### Cramp the Cascade

Managing CSS complexity and skipping the dreaded write-only CSS with SASS, focused components, naming conventions

Acknowledge

#### QoL Improvements

ES6, WEbpack, why you should put up with at least _some_ of the NPM world.

Acknowledgement of NPMs shortcomings.

Warnings of how to proceeed. Wathc out for the lack of Semantic versioning, or disregard for its truthfullness.

Warnings about excessive pollyfills. Watch out for the drug that is `npm install`. Beauty is in simplicity, aka, a slim package-lock.json

[^1]: The last thing you want is NPM to switch the owner of a package underneath you, and have compromised code deployed to prod because your CI pulled in something you'd never seen before. Every change to your frontend dependencies should go through thorough testing & code review, like all other code in your stack. This hypothetical, but 100% plausible post illustrates how bad it could get: https://hackernoon.com/im-harvesting-credit-card-numbers-and-passwords-from-your-site-here-s-how-9a8cb347c5b5 TODO: I recall reading about an actual package that had a malicious dependency installed via tricky version bumping, find that and reference