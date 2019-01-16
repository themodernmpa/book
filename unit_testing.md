## Unit Testing

Unit testing is a fundamental component of building a reliable application. Until recent years unit testing Javascript was heavy lift, but today there are a plethora of frameworks and tools to choose from.

### Karma & Jasmine

For the following examples we use Karma as test runner, and Jasmine as the test framework. We chose Jasmine because its syntax is very similar to RSpec, which we test our Ruby code with, and Karma because of its straightforward setup and simplicity in use.

Karma is a **test runner**, and is charged with maintaining the test environment, running your individual tests, and reporting the results. Karma gives a lot of nice add-ons, like IDE integration, remote test running across multiple client devices and CI environments, and auto-execution on file change.

Jasmine is the **test framework**, and gives you the tools you need to write your actual tests. It will typically provide hooks like `before`, and `after`, an interface to define mocks and stubs, and a nice syntax to define your tests with like `expect(someVal).toEqual(otherVal)`.

For running our tests, we integrated Karma execution directly into our CI pipeline. Any JS unit test failures will fail the entire build just like any RSpec failures. Our tests run for every PR, on every commit to an open PR, prior to any deployments to Staging, and immediately after any deployments to Production from Staging.

### JS Testing Lifecycle

If you haven't written JS unit tests before, it's important to understand what is going on under the hood. Fundamentally your tests are being run on an HTML page, in a headless browser. Your tests are isolated via an iframe. The parent page will load your tests, one by one, into success iframes. Each frame will execute its test, and report the results up to the parent page, which are in turn reported down and to your console.

There's no magic going on here, only a sequence of steps, and it's important to know that all your code is fundamentally running in a browser. At its heart your unit tests are executing chunk of code on its own HTML page.

### Stimulus Testing Support

As mentioned in the [Stimulus chapter](stimulus.md), the Stimulus lifecycle triggers on the addition or removal of a DOM node via a `MutationObserver`. In our minds this makes the DOM associated to a Stimulus controller an *essential portion of the controller*. As the Stimulus controller will never be initialized without its associated DOM, we saw no reason to attempt to test the component without its DOM.

This made the theoretical test lifecycle identical to its real-world lifecycle, which also means we don't have all that much to do to get some basic tests working, at its core we need to initialize a controller and bind it to some DOM.

### App Helper

To make the Stimulus lifecycle work as it does in production, we need to initialize a Stimulus app. We'll be initializing this app in every controller test, so we wrote a helper and tucked it away for recycling.

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

### A Simple Test

```javascript
// spec/counter_controller_spec.js
import CounterController from "../../frontend/controllers/counter_controller";
import { testApp } from "../spec_helpers/stimulus";

describe("CounterController", () => {
  const identifier = 'counter';
  const template   = `<div data-controller="${identifier}"><button data-action="click->counter#increment" data-target="counter.count">0</button></div>`
  let app;

  beforeAll(done => {
    app = testApp();
    app.register(identifier, CounterController)
    done()
  })

  beforeEach(done => {
    const shell = document.createElement('div')
    shell.innerHTML = template
    document.body.appendChild(shell)
  })

  
  it("does not increment on mount", () => {
    const target = document.querySelector('[data-target="counter.count"]')
    expect(target.innerText).toEqual('0')
  })
  
  it("increments the counter on click", () => {
    const clickTarget = document.querySelector('[data-action="click->counter#increment"]')
    clickTarget.click()
    const valueTarget = document.querySelector('[data-target="counter.count"]')
    expect(target.innerText).toEqual('1')
  })
})

```

While this example is contrived and trivial, it demonstrates that testing your Stimulus components can be very straightforward. You mount the HTML, and then issue actions against it and measure the results. 

There are areas for improvement. DOM selection will get tedious here, but that is trivial to clean up with a few helper functions.

In our experience the most difficult part of growing this solution is keeping your test HTML in sync with your application HTML. As you iterate your application it's quite easy to update the HTML that powers your Stimulus component, but forget to go and update your test suite as well. Often this won't even cause a test failure until the HTML fragments are so out of sync that it becomes a major pain to update them. If you're not careful this can result in a raft of abandoned tests.

Additionally, as your application grows in complexity so will your Stimulus components. It can be a challenge to perpetually sync HTML for a component that spans 30+ lines of HTML. Even if you take a shortcut and only sync the elements that are assigned Stimulus actions and targets, you're still left with a tedious task, and now your tests are officially diverging from your production code.

In our last project we solved this problem by abusing RSpec, taking advantage of Rails 5.2's new `ApplicationController.render()` method, and a few conventions (as is the Rails way).

### Generating Test HTML

The following section is going to be very Rails-specific, but we hope that the examples can provide value to you no matter your language and ecosystem.

As our last application grew it became apparent that the task of hand-writing HTML in our unit tests was killing the motivation to write tests for all but the most mandatory cases. As we were also using HAML as our HTML preprocessor, the task was even more onerous. It became obvious that we needed an automated solution. We sat down and produced a short list of requirements to ease the pain:

- Controller HTML needed to be programmatically generated; we wanted to banish the idea of manually writing HTML in our unit tests.
- Because many of our Stimulus controllers contained conditional branches in `connect()`, we would also need the ability to generate multiple variations of the same fragment. 
- We needed an easy way to load HTML fragments on disc, and we needed the ability to conditionally mount these fragments in our test suite.
- We wanted our fragments to be transient. Every test run needed to regenerate all fragments so that we would sidestep errors resulting from tests being out of sync with production.

##### Abusing RSpec to generate fragments

Rails 5.2 introduced the method `ApplicationController.render()`, making it easy to render an arbitrary template outside of the context of a controller. This quickly became the core of our technique. The following module was written, and imported into the RSpec suite.

```ruby
module FixtureRendering
  def renderFixture(haml:, assigns: {}, locals: {}, variant: '')
    content = ApplicationController.render(template: haml, layout: false, assigns: assigns, locals: locals)
    fixture_path = Pathname(Rails.root.join('spec/fixtures/frontend', "#{haml}#{variant}.html"))
    fixture_path.dirname.mkpath
    fixture_path.write(content)
  end
end
```

This method allows us to define a HAML file, a variant, and pass in the `assigns` and `locals` params needed to set up all needed state to render any kind of template.

It writes out the generated file to `app_root/spec/fixtures/frontend`. We write out the full path of the HAML template, allowing us to maintain identical directory structures between the frontend fixtures folder and the Rails `app/views` controller.

Finally, we needed a convention by which to execute this method. We should note that we have all of our Karma unit tests located in `app_root/spec/frontend`. This is technically inside our RSpec test suite, but since RSpec doesn't run javascript files, it was a convenient way to organize all tests in one location. Suddenly, this decision also made the next step trivial. We decided to simply use an RSpec test to generate our fixtures. Here is a simple example:

```ruby
# ./spec/frontend/controllers/dialer/status_controller_fixtures_spec.rb
require 'rails_helper'

RSpec.describe 'status_controller fixture generation', type: :fixturegen do
  describe 'status.html' do
    let(:user) { create(:user_with_ownership) }
    let(:conversation) { create(:conversation, user: user) }

    it 'generates a fixture for outgoing calls' do
      renderFixture haml: 'dialer/_status',
                    locals: {
                      conversation_id: nil,
                      formatted_number: '+1 (123) 123-1234'
                    }
    end

    it 'generates a fixture for incoming calls' do
      renderFixture haml: 'dialer/_status',
                    variant: '-incoming',
                    locals: {
                      conversation_id: conversation.id,
                      formatted_number: '+1 (123) 123-1234'
                    }
    end
  end
end

```

This is what we mean by "Abusing RSpec". Technically this spec does not assert anything, instead it exists solely to generate some HTML. Note the file location at the top, this spec file lives side by side with its JS unit test in the `spec/frontend` folder. We found this organization very convenient.

Because we are generating our fragments in RSpec, we have gained full access to our entire Rails application's runtime. We have the ability to stub and mock anything, we have access to all of our models and service objects, and most importantly, *it is now trivial to generate any possible HTML fragment variation*.

##### Generating variations of the same fragment

Many of our Stimulus controllers contain branching logic in `connect()` based on what data attributes are present and how they are populated. This meant that we needed the ability to define these variations during fragment generation, and that one HAML file in our application could result in multiple HTML files in our fragments folder.

Because we're able to pass our `locals` and `assigns` params right through to the Rails renderer, it's trivial for us to send in the needed data to meet any variation requirements inside the HAML template. With the addition of the `variant` key, we are also able to append a variant name to our resulting HTML fragment. The fixture generator above produces two files:

```
./spec/fixtures/frontend/dialer/_status.html
./spec/fixtures/frontend/dialer/_status-incoming.html
```

We do caution against generating too many variations of a single template, this can be a smell indicating that you're not testing a small enough unit of code, or perhaps your Stimulus controller is simply doing too much, and should be split into multiple controllers.

##### Easy loading of fragments from disc

Now that we had HTML fragments being generated, we needed to pull them into Karma. This is an area where we were more than happy to lean the existing NPM ecosystem. The combination of the packages `karma-fixture` and `karma-html2js-preprocessor`, and some minor configuration allowed us to refactor the example JS test from above into the following:

```javascript
// spec/counter_controller_spec.js
import CounterController from "../../frontend/controllers/counter_controller";
import { testApp } from "../spec_helpers/stimulus";

describe("CounterController", () => {
  const identifier = 'counter';
  let app, controller;

  beforeAll(() => app = testApp())
  beforeEach(done => {
    app.register(identifier, CounterController)
    fixture.setBase('')
    fixture.load("counter.html")
    done()
  })

  // The expectations did not change with this update
})
```

Note that the inline HTML is gone, and instead we are using `fixture.load()` to manage inserting the needed HTML into our test suite. Done!

##### Keep the fragments transient

Keeping the generated HTML transient was the easiest part of setting up this system. We added one line to our `.gitignore`, and double-checked to make sure that in our CI pipeline our Karma tests ran _after_ our RSpec tests. With these in place, every CI run would come down without any HTML fragments, and they would all be generated during the natural RSpec run. Assuming RSpec succeeded, our Karma tests would then execute, referencing the freshly generated HTML.

### Testing Async Actions

You may notice the use of `done()` in the last code snippet. For those not familiar with Jasmine, `done()` is how you tell the test suite that "I'm done issuing async actions". Jasmine will wait at `done()` before finalizing your test assertions. If you set timers or make async requests you'll want to read the docs for `done()`, as it can be one of the tricker parts of unit testing javascript.

On the topic of making async requests, we **highly** recommend that you invest in a mocking library like `fetch-mock` or similar. Your unit tests should never reach outside of their local environment. You really don't want to have to run a web server just to serve back test data to your frontend application to assert success on, that is a recipe for tears and gnashing of teeth.

Instead, prefer to write fixture files defining the response data, and use a mocking library to highjack `fetch` or `XMLHttpRequest`.

### Conclusion

As of Stimulus `v1.1.0`, Stimulus itself does not have an official recommendation for unit testing controllers, nor does it provide any official test bootstrapping. However, we hope that you see that while this chapter is fairly long, there are not that many actual steps taken. Once we made the realization that a controller's DOM is part of the controller *unit*, and that generating the DOM via our existing means would make life easier, all other parts fell together.

We are now able to build contexts and expectations for every variation required. Because the HTML for our tests is generated in the same location that generates the same HTML for production, we no longer suffer from drift between test HTML and running code. By simulating user interactions with our HTML fragments, we are able to assert that any logic contained in a Stimulus controller is executed appropriately.

