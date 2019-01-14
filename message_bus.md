## Communicating between Components

As you start adding Stimulus controllers to your application, eventually you're going to run into a situation where you want 2 controllers to talk to one another. A user selects a date in `DatepickerController`, and you want `DataDisplayController` to respond to this selection by issuing a new `fetch` request to refresh the display.

Cross-component communication is a critical part of keeping your controllers small, and specialized. While you could make your `DataDisplayController` handle your datepicker bindings, doing so will prevent you from easily recycling your datepicker code to other, similar displays. Eventually you'll need to add other forms of data filtering, further expanding the responsibilities of the `DataDisplayController`. 

We will solve this problem with an `Event Bus`. You're already familiar with events, they're a fundamental part of building any JS-based application. We'll start with the basics, and flesh out a few nice-to-haves,  making publishing and subscribing easy.

### Event Bus Basics

At its core, this event bus is a simple wrapper around the following code:

```javascript
// DatepickerController.js
const payload = { date: selectedDate }
const event = new CustomEvent('datepicker:selected', { detail: payload })
this.element.dispatchEvent(event)

// DataDisplayController.js
const selectedDateHandler = (event) => {
  console.log(`Datepicker selected date: ${event.detail}`)
}
document.body.addEventListener('datepicker:selected', selectedDateHandler)
```

Here we're publishing a custom event with a custom name of `datepicker:selected`, and a payload containing the date selected. We're publishing the event on the Stimulus controller element, allowing it to bubble up the DOM tree. Elsewhere we are listening for the custom event name on the `body` element, and the handler is logging out the published date.

If you have only a few components in your project, this is probably all you need to enable cross component communication. It's simple and easy to understand, and if you don't need more we encourage you to go with the simplest thing that works.

Since we knew that we would need cross component communication all across our app, we will take a few more steps.

### Formalization Goals

Publishing an event is simple enough, but when you know you might have dozens, if not hundreds of components living in an application it helps to formalize how you are publishing and subscribing. Along the way we can also prevent some common bugs by creating a few conventions.

1. Avoid duplication with a canonical way to publish & subscribe
2. Avoid magic strings & typo-based bugs with defined event names
3. Enable accurate telemetric logging with default data
4. Enable dead-simple Stimulus integration by enhancing our `ApplicationController`

### A canonical way to publish and subscribe

Take our above simple example and formalize it into a standalone module.

```javascript
// ./lib/event_bus.js
export const events = {
  DATEPICKER_PUBLISH: "datepicker:publish"
}

const subscribe = (eventName, callback) => {
  if (!events[eventName]) {
    throw `Cannot subscribe to undefined event ${eventName}`
  }
  if (!callback) {
    throw `Cannot subscribe to ${eventName} with an undefined callback`   
  }

  document.body.addEventListener(eventName, callback)
  return () => document.body.removeEventListener(eventName, callback)
}

const publish = (eventName, data = {}) => {
  if (!events[eventName]) {
    throw `Cannot publish to undefined event ${eventName}`
  }

  data.sent_at = new Date().toISOString()

  const event = new CustomEvent(eventName, {
    detail: Object.assign({}, data),
  })
  document.dispatchEvent(event)
}

export const bus = {
  publish: publish,
  subsubscribe: subscribe
};
```

There are a few interesting details here not seen in our first example. The first are all the checks against existence of the subscribed & published event names. This is a protection mechanism designed to save us from typos and "magic strings". You never want to have your frontend code littered with `datepicker:select` all over the place. It's too easy to write `datepicker:selected` and end up with a bug. Even worse is to end up with similar but disparate events and _no_ bug. Over time you'll add more functionality or data to one event, and not to another, and eventually someone will have to figure out which event is canonical.

We export the `events` object so that in our individual components we can import the object and reference predefined events. This convention makes it easy to enforce the No Magic Strings rule.

Another interesting change here is the addition of a date string to every event payload under the `sent_at` key. If you log out all your event publishing to the console, you can pick it up via a telemetrics system, making debugging far easier.

Lastly, take note of the returned data in the `subscribe()` method:

```javascript
return () => document.body.removeEventListener(eventName, callback)
```

Here we are returning a function that, when executed, will remove the event listener we've added. We've found this pattern to relieve a lot of mundane tracking. You won't need to memoize the callback function when you subscribe, you will be given an unsubscribe function that you can memoize instead. We'll use this to our advantage in a little bit.

### Integrating with Stimulus

Now that we have a single point of pub/sub for our application, we can integrate this directly into our Stimulus controllers with a couple methods added to our base controller.

First the code, then we'll walk through it.

```javascript
// application_controller.js
// TODO: Resolve the name & location of this file with the file mentioned in Application Structure
import { Controller } from "stimulus";
import { bus, events as busEvents } from "../helpers/eventbus";

export const events = busEvents

export class ApplicationController extends Controller {
  initialize() {
    this.subscriptions = []   
  }
 
  publish(event_name, data) {
    bus.publish(event_name, data);
  }
  
  subscribe(event_name, callback) {
    const subscription = bus.subscribe(event_name, callback);
    this.subscriptions.push(subscription);
  }

  unsubscribe() {
    this.subscriptions.forEach(unsub => unsub())
  }

  disconnect() {
    this.unsubscribe();
  }
}
```

As you can see, this is another thin wrapper around an already thin wrapper, around a simple concept. Not too much going on here, right? We import the event bus, and then we add a few functions to our `ApplicationController` to enable easy access to the bus. There are a few key points here we should highlight.

In `initialize()` we create an internal property named `subscriptions`, an array. This is where we store the unsubscribe functions that the event bus returns when subscribing to an event. This in turn enables us to automatically unsubscribe from these events when `disconnect()` is called. 

Lastly, as a convenience in our other Stimulus controllers, we both import the `events` object and re-export it from the application controller. This is done to minimize the amount of importing we need to do down the line. In most cases we will already be importing the application controller, so we save ourselves a line in multiple files by enabling multiple imports.

This thin abstraction in our `ApplicationController` solves a few gotchas for us.

1. Orphaned event listeners are no longer a thing, as events are unsubscribed from when the Stimulus component disconnects from the DOM. Unless you need to override an individual controllers `disconnect()` function you don't really need to think about orphaned listeners.
2. Verbose subscribe and unsubscribe is eliminated, in most cases you will simply call `this.subscribe` with a given event name and a handler.
3. We've effectively eliminated bugs due to magic event names, as the bus will throw errors when an undefined event is used. 
4. We've enabled simple cross-component communication using events, which every web developer already knows how to work with, and we've tucked the implementation details away from our day-to-day usage while keeping it available for other non-stimulus components to use.

### Integrating with our Use Case

Returning to our use case where we have a `DatepickerController` controlling a date picker, and a `DataDisplayController` that we want to make respond to the change in dates.

```javascript
// datepicker_controller.js
// let us assume this controller is bound to a simple input field
import { ApplicationController, events } from "./application_controller"

export class DatepickerController extends ApplicationController
  connect() {
    this.element.addEventListener("change", event => this.changeDate(event))
  }

  changeDate(event) {
    this.publish(events.DATEPICKER_SELECT, { date: event.targetElement.value })
  }
}
```



```javascript
// data_display_controller.js
import { ApplicationController, events } from "./application_controller"

export class DataDisplayController extends ApplicationController
  connect() {
    this.subscribe(events.DATEPICKER_SELECT, data => this.dateSelected(data))
  }

  dateSelected(data) {
    // respond to the date update event by refreshing the data display table
  }
}
```

### Downsides

As with every solution to any problem, this isn't perfect and there are some downsides.

Every event here is synchronous*. Subscribers `CustomEvent` are executed one after another, and unlike native DOM events there is no interleaving of execution if one `CustomEvent` takes longer than another. This can cause some blocking issues if you do a lot of heavy lifting in event listeners. The easy solution is to make every event execute async via `setTimeout`, but we will leave that as an exercise to the reader.

Another downside is maintaining event payload consistency. Over time you may begin to publish different types of data under the same event name, which can lead to odd bugs where one event works, and another doesn't, depending on who is publishing and who is subscribing. 

TODO: TJ: I had the idea of adding a simple data payload validator to the event bus, but we never really hit the point where it would be necessary. But the basic idea would be to have a 2nd object, complementary to `events`, where the key is the event name and the value is an array or object containing object keys, and on publishing event we look up the key list and ensure the payload object has each appropriate key. No tests for values, just a simple "object must be in this form".

### Testing

One advantage of publishing events for cross-component communication is that you do not need to test your two components together, as you now have a defined boundary. You know that *under action X, event Y is published*, and you can unit test that condition. You don't need to worry about unit testing that a `fetch()` request from `DataDisplayController` is issued when action X is performed against `DatepickerController`, instead you write 2 unit tests. One that asserts `with action X, expect event Y`, and another that asserts `when event Y, expect action Z`.