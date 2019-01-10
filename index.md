## The Modern Multi Page Application

You probably don't need a Single Page App after all.

1. [Cross Component Communication](message_bus)
2. [Appication Structure](application_structure)
3. [Multi Page App vs SPA](mpa_vs_spa)
4. [Preface](preface)
5. [Stimulus](stimulus)
6. [Toolset](toolset)
7. [Turbolinks](turbolinks)
8. [Tying it all Together](tying_it_together)

Modern web applications are ultra responsive, fast to load and visually rich - our customers expect snappy response times and repeated research has shown the adverse impact of web pages that load too slowly [^1].  Browsers however have traditionally been a poor platform for delivering these experiences.  (more color needed)

Enginers are used to working around problems with browsers - one of the most ubiquitous Javascript tools - jQuery - was born out of a need to have a single API for interacting with browsers that all behaved differently.  We are acustomed to not rely on the browser and to shy away from the request/response paradigm using tools and frameworks to keep development "easy" while supporting a variety of runtimes. 

As browsers became increasingly consistent and accurate in how they implement the Javascript and CSS specificications, the need for framework level abstractions was reduced and we saw the death (in practical terms) of frameworks like ExtJS and YUI.  At the same time browsers were becoming more consistent, our customers and our business partners were asking for more and more complex applications with faster response times and mroe rapid deployments.  To push the envelope, engineers developed other techniques for working around browser limitations (namely slow request/responst times) and for reusing code on the client side. In the early 2010s we saw these workaround coalece into a front end architecture known as a SPA or Single Page App which became the reference architecture for modern web applications.  

The goal of a SPA is create a better user experience by working around browser indiosyncricies, reducing the amount of information that has to be exchanged when changes are made and to provide a mechanism for updating parts of a web page without reloading the page.  SPAs perform this job well, they can be quite fast, highly modular (for testability) and compossible.  However, they are also quite complex and introduce significant friction on the software development lifecycle due to the engineering burden required to create and maintain them.  Additionally, the lifecycle of SPA frameworks is nearing it's peak - browsers continue to become more consistent in their APIs and functionality while at the same time  introducing support for features like modules within the browser and HTTP2 which will make many code compilation tools and abstractions unnecessary in the future. 

This book was written to showcase how "traditional" web development techniques can be used to create a modern Multi Page Application that is much faster to build and maintain while providing a user experience that delights your customers.  This book illustrates a reference architecture for frontend applications that emphasises speed of development and product/feature iteribility while providing many/most of the benefits of more complex architectures.  This approach brings into alignment the need of the business to release features and quickly get customer feedback while letting the engineering team build features with the right level of rigor and "correctness" for the business case at hand.



When might this not be for you?



- **01 - Preface** - Who is this book for?
- **02 - The Problem At Hand ** - Why do we reach for a SPA?
- **03 - The Technologies Involved**
  - Being Vanilla, introduction to the concepts we need to roll our own solution.
  - A Persistant JS Process -- Turbolinks introduction
  - Skip the jQuery Soup -- Stimulus introduction
  - Cramp the Cascade -- Prevent write-only CSS with SASS
  - Quality of Life Improvements -- ES6 via Webpack
    - Some things that are nice about webpack, modules, imports
    - Why you shouldn't go overboard with your polyfills, keep it reasonable.
- **04 - The Web App Lifecycle**
  - Short overview of the standard web app lifecycle w/jQuery sprinkles
  - The Turbolinks Lifecycle
  - The Stimulus Lifecycle
  - BASE - Marrying Turbolinks & Stimulus with a Base controller & event hooks
  - Your new app lifecycle
- **Respecting the URL**
  - A list of reasons about how important the URL is, and how important it is to respect it and _use_ it.
  - Store data in it! Use Turbolinks to update it if a major component could resume state from it!
- **05 - Thinking in Components**
  - Component mount, dismount, teardown, pre-cache techniques for smooth page transitions
  - Don't go outside components scope! Nonos regarding scope and area of domain.
  - Storing State in the DOM vs in the Component
    - The DOM should always reflect what could come from teh server on a refresh
    - The component should track what happens after the refresh
  - Singleton Components with data-turbolinks-permenant
  - Warning - Too many operations in `connect()`
- **06 - Inter-Component Communication**
  - The situation -- As your app grows, suddenly one component needs to know what another component did _without_ a page refresh.
  - Reminder about not reaching out of scope.
  - Direct component communication via the Stimulus app instance.
  - Events, publish a DOM event and listen on parent elements & the root.
  - Gotchas with Events.
  - 



[^1]: https://www.fastcompany.com/1825005/how-one-second-could-cost-amazon-16-billion-sales and http://glinden.blogspot.com/2006/11/marissa-mayer-at-web-20.html

