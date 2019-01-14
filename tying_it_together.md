## Tying it Together

While a traditional SPA framework gives you a standardized way to boot up your application, when developing a web app using a collection of libraries you will have to build out this structure yourself. By default Turbolinks needs no initialization, and Stimulus requres a small amount.

As mentioned previously, we're going to use Webpack as our asset manager of choice. However the following can be used with any asset pipeline. You could even go vanilla JS via the script `type="module"` attribute, if you have luxury of targeting only evergreen environments.

Let's start by defining a directory structure four our frontend assets. By convention we use a root folder named `frontend`, in which all of the following are contained:

```
./application.js # Our root application file
./controllers # Stimulus controllers are stored here
./stylesheets # CSS files go here
./helpers # JS helper files for trivial and commonly used code
./lib # Any non-Stimulus, non-trivial JS goes here
./packs # Webpack entry files, anything inside here will become an output file
```

### packs/*.js

Any files stored in `./packs` will be consumed by Webpack, and will result in a generated file that you can link to in your application. We prefer to do as little as possible here. We consume our root application file, all Stimulus controllers, and all (S)CSS files. Because Stimulus triggers functionality on the presence of a DOM node, we are able to import only what is needed for each controller, which has a cascade effect of only importing what is used through our application.

```javascript
// Imports the main app boot file
// TODO: Validate this. We currently import our boot file as a byproduct of it being included in a couple of controller files.
import "../application"

// Import all Stimulus controllers.
import "controllers"

// Imports all stylesheets
// Stylesheet import produces a separate css file due to our configuration.
// TODO: Validate that we don't have a plugin enabling this functionality. I didn't see one but I recall installing or configuring something to get this to work properly.
require.context("../stylesheets/", true, /\.scss$/)
```

The result of this small file will be 2 output files, `application.js` and `application.css`

### ./application.js

This is where we will initialize our Stimulus application, register controllers, and tie our controllers into the Turbolinks lifecycle with a a Base Controller and event hooks. There will be a fair amount going on here, so first, the code, and then we'll walk through each section in turn.

```javascript
import { Application, Controller } from "stimulus"

// Boot the stimulus application
export const application = Application.start()

// Bind Turbolinks events to controller methods
document.addEventListener("turbolinks:click", () => {
  application.controllers.forEach(controller => {
    if (typeof controller.teardown === "function") {
      controller.navigation()
    }
  })
})

document.addEventListener("turbolinks:before-cache", () => {
  application.controllers.forEach(controller => {
    if (typeof controller.teardown === "function") {
      controller.teardown()
    }
  })
})

// Handle errors generated inside Stimulus controllers
application.handleError = (error, message, detail) => {
  console.error(message, error, detail)
}

// Define a base controller for us to exend elsewhere
export class ApplicationController extends Controller {
  navigation() {}
  teardown() {}
}

```

### Booting the application

```javascript
import { Application, Controller } from "stimulus"

// Boot the stimulus application
const application = Application.start()
```

This is a simple import and initialize, directly from the Stimulus docs. You could export the application constant here if you need to, but in our experience if you're reaching into your Stimulus application from external modules, you may want to rethink your approach.

### Marrying Turbolinks & Stimulus

```javascript
// Bind Turbolinks events to controller methods
document.addEventListener("turbolinks:before-cache", () => {
  application.controllers.forEach(controller => {
    if (typeof controller.teardown === "function") {
      controller.teardown()
    }
  })
})
```

This is our favorite part. While Stimulus automatically fires a nice sequence of `connect()` and `disconnect()` methods as DOM enters and exits, Turbolinks offers an additional series of events that, when tied into your Stimulus controllers, gives you a rich lifecycle with which to manage your frontend components.

At a minimum you should tie `turbolinks:before-cache` to a `teardown()` method. The Stimulus `disconnect()` method will fire when a controller exits the DOM, but this will be _after_ the DOM is removed from the current document. It will be too late for you to remove any bindings or do any cleanup of your component before Turbolinks caches your page DOM.

Attaching a mehtod to the `turbolinks:click` event is also recommended if you want controllers to gain the ability to cancel a navigation event. Consider a controller tied to a comment system; instead of binding to `document.unload` to prompt the user with a confirmation dialog, you could issue this dialog in a navigation()` lifecycle method.

Note that in the event listener we are conditionally executing our custom lifecycle functions. You may not always inherit from your base controller, but all controllers in your app will be available registered within `application.controllers`, and are iterated over in the event handler.

### Handling Errors

```javascript
// Handle errors generated inside Stimulus controllers
application.handleError = (error, message, detail) => {
  console.error(message, error, detail)
}
```

This method mirrors the Stimulus default, catching errors and logging them out to the console. In our production applications this is where we would capture and log errors to our application error reporter. Or you can leave it as-is, this really depends on how you handle errors.

### A Base Controller

```javascript
// ./controllers/application_controller.js
// Define a base controller for us to exend elsewhere
export class ApplicationController extends Controller {
  navigation() {}
  teardown() {}
}
```

A base controller, defined here as `ApplicationController`, is the root of your frontend app. We extend almost all components off of it, so it is the default place to add global functionality, which we will show examples of later.

Here we are defining empty lifecycle events. Because our Turbolinks -> Stimulus binding is conditional on the lifecycle event being present, it's not necessary to define these methods, but we like to keep our code explicit if possible.

