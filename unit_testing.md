## Unit Testing

Unit testing is a fundamental component of building a reliable application, no matter the language or deployment target. We find frontend unit tests particulairly useful for debugging problems, and the act of leaving a test written for flushing out a bug doubles as a regression test, ensuring the bug does not reappear. Until recent years unit testing Javascript was heavy lift, but today there are a plethora of frameworks and tools to choose from.

### Karma & Jasmine

For the following examples we use Karma as test runner, and Jasmine as the test framework. We chose Jasmine because its syntax is very similar to RSpec, which we test our Ruby code with, and Karma because of its straightforward setup and simplicity in use.

A **test runner** is charged with maintaining the test environment, running your individual tests, and reporting the results. Karma gives a lot of nice add-ons, like IDE integration, remote test running across multiple client devices and CI environments, and auto-execution on file change.

The **test framework** gives you an expectation syntax you like to use. It will typically provide hooks like `before`, and `after`, an interface to define mocks and stubs, and a nice syntax to define your tests with like `expect(someVal).toEqual(otherVal)`.

For running our tests, we integrated Karma execution directly into our CI pipeline. Any JS unit test failures will fail the entire build just like any RSpec failures. Our tests run for every PR put up, on every commit to an open PR, prior to any deployments to Staging, and immediatly after any deployments to Production from Staging. By convention we do not deploy any Staging builds with test failures, but we do not prevent these deployments on the off-chance that we have to do an emergency deployment through a minor test failure.

### JS Testing Lifecycle

If you haven't written JS unit tests before, it's important to understand what is going on under the hood. Fundamentally your tests are being run on an HTML page, in a headless browser. Your tests are isolated via an iframe. The parent page will load your tests, one by one, into succes iframes. Each frame will execute its test, and report the results up to the parent page, which are in turn reported down and to your console.

There's no magic going on here, only a sequence of steps, and it's important to know that all your code is fundamentally running in a browser. At its heart your unit testings are executing chunk of code on its own HTML page.

### Stimulus Testing Support

As of Stimulus `v1.1.0`, Stimulus itself does not have an official recommendation for unit testing controllers, nor does it provide any official test bootstrapping. Since the lifecycle of a Stimulus controller is so straightforward, we wrote some glue code on our own to make testing pretty easy.

As mentioned in the [Stimulus chapter](stimulus.md), the Stimulus lifecycle triggers on the addition or removal of a DOM node via a `MutationObserver`. In our minds this makes the DOM associated to a Stimulus controller an *essential portion of the controller*. As the Stimulus controller will never be initialized without its associated DOM, we saw no reason to attempt to test the component without its DOM.

This made the theoretical test lifecycle idential to its real-world lifecycle, which also means we don't have all that much to do to get some basic tests working.

1. Boot your Stimulus application & register your controller
2. Append your controller DOM
3. Assert any desired `connect()` functionality has taken place
4. Trigger any desired user actions and assert their side effects
5. Trigger any desired events and assert their side effects
6. Remove your DOM from the page
7. Assert any `teardown()` or `disconnect()` events have taken place
8. Stop your application

8 steps looks like a lot, but we've found that for most controllers you will only need a subset of these steps. It really depends on what your controller does.

### An Example Test

As we mentioned, we wrote some custom glue code to assist in testing. The first is a `testApp()` method. We tuck this away in a test helper file and import into each test.

```javascript
// spec/stimulus_helpers.js
import { Application as StimulusApp } from "stimulus";

export const testApp = () => {
  const app = StimulusApp.start();

  app.handleError = (error, message, detail) => {
    let err = `${message}: ${error.toString()}`
    throw err
  }

  return app;
}
```

Two important things to note here. First, this method returns the initialized application, we'll need this object available elsewhere for registering controllers. Second, we define a custom error handler and intentionally `throw` any caught errors. You can configure Karma to consider `console.error` a test failure if you wish. We prefer console logging to be informational, not actionable. 

Also of note is that we're not importing the application constant defined for our frontend application. We're looking to unit test a single controller, so we will have to fire any custom lifecycle events ourselves.

```javascript
// spec/counter_controller_spec.js
import CounterController from "../../../frontend/controllers/counter_controller";
import { testApp, attachTemplate, removeTemplate } from "../spec_helpers/stimulus";

describe("CounterController", () => {
  const identifier = 'player';
  const template   = `<div id="testController" data-controller="${identifier}"></div>`
  let app;

  beforeAll((done) => {
    app = testApp();
    app.register(identifier, PlayerController)

    done()
  })

  afterAll((done) => {
    app.stop()
    done()
  })

  beforeEach((done) => {
    attachTemplate(template)
    done()
  })

  afterEach(() => {
    removeTemplate()
  })

  function controllerDOM() {
    return document.getElementById("testController")
  }

  it("mounts", () => {
    expect(controllerDOM().dataset.controller).toEqual(identifier)
  })
})

```



. Many times Unit tests tell the next programmer how a piece of code is supposed to function, they ensure that the piece of code works as intended

- Karma & Jasmine setup
- Helper functions for simple testing
- Generators
- DOM generation with RSpec
