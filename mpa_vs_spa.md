## The Problem At Hand

Skip this chapter if you're looking for technical nitty-gritty, or if you're already decided on the how's and why's of an MPA for your use case. The following fluff probalby won't benifit you at all.

Skip this chapter if you're not in the mood for some snark. Some of this chapter is a bit of cathartsis for me.

Before we start digging into the hows and why's, let's define a few assumptions.

#### MPA - Multi Page Application

The MPA, or Multi Page Application, is your typical request-response web application. Nearly all websites and web apps are MPAs. MPA is the default state of the web. Let's look at a series of example URLs

`http://example.com/contacts/1234/outreach`
`http://example.com/campaigns/4321/update`

In an MPA, if you were to execute these requests and trace the request through the application, you will see the request hit a routing interface, be dispatched to a controller (`/contacts => contact_controller.rb`, `/campaigns => campaigns_controller`, etc), which in turn will execute business logic, render a template, and return a response. 

You'll be able to trace an entirely separate code path for each route *in your web application server*. You'll typically serve a fully-formed HTML page back to the client as a response.

#### SPA - Single Page Application

A SPA, or Single Page Application, is named as such because the backend web-server will typically serve only a single actual page. Looking at the same example URLs, for requests to `/contacts`, the request will hit the web server, and be routed to a single internal controller, let's call it `app_controller.rb`. This controller will then execute _no_ business logic [^1], instead it will respond with a nearly-empty HTML response, which will contain a reference to the SPA asset pack. The browser will download this asset pack and execute the contained JS, which starts up our SPA.

The SPA will boot up, grab the URL, and see if it has any mappings that handle `/contacts`. If it does, it'll execute the handler for `/contacts`, which in turn will execute an Async request to fetch data for contact ID `1234`, which in turn will hydrate some internal data store, then render whatever needs to be rendered.

At this point any subsequent page transitions will no longer get routed down into `app_controller.rb`. It's done, it has served its single page. All future click events are hijacked by JS, and pipe through the aformentioned URL mapping process. Mappings are deduced, requests are made, datastores are hydrated, pages are displayed, but the web server has done its job. [^2]

#### What is a SPA good at?

SPAs are self-contained. The dream is a single JS file that you can drop on a CDN somewhere, embed into a mobile app, reference in an iframe[^5], whatever. Once your user facing app is standalone, a lot of options open up for you.

SPAs are a separate application, . You can isolate your JS developers from your API developers. [^4]By isolating your JS developers, you dream that they can start iterating faster. Visions of daily interface deployments dance in the eyes of project managers everywhere. As soon as those grumpy DB developers are out of the loop *we'll show them*. UI iteration will save the day!

SPAs give you full and insanely granular control your frontend application. Do you want to track all mouse movement, keypresses & focus events all the time? Do you want to record everything your users do, ship it off somewhere, and enable a replay feature whereby your employees could rewatch every user session as if they were standing over the shoulder of your users? [^7] A SPA really helps here.

Many use cases really benifit from a SPA. Are you building an Adobe Photoshop clone? Probably should use a SPA. Are you building a web game? Probably use a SPA. [^6]

#### What is an MPA good at? 

MPA is pretty much the default state of a web application. It's kind of hard to say what it's _good_ at, because it does it all, and it can be built and customized to be really, really good at most things. Instead of looking at what an MPA is good at, let's look at what they're _bad_ at.

MPAs are bad at maintining a persistant JS runtime. If you want to do complex actions across page loads, that can be impossible or obscenely dificult.

MPAs typically ship more data over the wire per page request. You're usually shlepping down a big DOM blob with every response, so over time the argument is you'll use up more user bandwidth, and each page load will take more time.

MPAs typically struggle with providing rich UI interactions that impact large areas of the page. An example would be an email client. Composing an email, sending it, updating the Sent item count, and adding a Sent message to a sidebar can be a UI transaction that's dificult to keep maintainable using the Javascript Sprinkles approach, expecially as more and more functionality gets glued into the "MessageSent" transaction.

#### OK, I think I need a SPA, the PM talks about doing...

If you *think you need* then you don't need. This isn't just a SPA vs MPA issue, this is a fundamental toolset issue. When you *think* you need, then what you *really* need is research, which will tell you what you *truly* need.

Everything you add to your app stack adds complexity. Complexity is the black death of this industry, you should be more allergic to complexity than you would a zombie shambling after you for your brains. Don't add *anything* to your tech stack until you absolutly need to. Adding a SPA is adding another separate application. You should not underestimate the amount of complexity this adds to your life, your initial estimations will be off double, at least.

You've probably seen this in other areas of managing your web application. Your generic search was slow, so you introduce ElasticSearch, and now it's fast but you've really increased the workload on your devops team, and there's a shitton of new code managing search. Hell, you might even have an engineer in the team who is now looking for a new gig, because everyone ignored her when she said that she could fix the search queries if given a week or two [^8], or even build for the future if given a month.

Your job as the technical person in the room is to loudly and clearly illustrate how bad complexity is. It is trivial for anyone who won't have to do the work to underestimate how much they're asking of you. If you're a black box of Make It Happen, you will ultimately end up with PM after PM shoveling hopes and dreams right up your JIRA, and believe me that's very uncomfortable.

If you look at other teams (particularily non-technical teams) and think they're a bunch of whiners who seem to be successful at dodging "the hard work" by complaining, and your technical teams are always drowning under unreasonable demands but you're proud of how you "get it done", you should consider the otpion that you are the problem. You're not effectivly communicating complexity, you're making it too easy for people who *just don't know* to ask for too much. The organization is making critical decisions without full information!

#### Enough Snark, whaddaya got?

If you're still with me here it can be easy to think I'm just a bitter traditional dev who isn't onboard with the new hotness. Sure, that's kind of true, but what's more true is how much I believe in *The Right Tool For The Job*, and how much I believe that a majority of our industries new developers *don't know what's available to them*. 

Let's start looking at the good stuff. How can we mitigate the things MPAs are bad at, persistant JS processes, rich UI interactions, complex multi-component transactions, while maintaining what they're good at; velocity, dead-trivial CRUD operations, fast on boarding, ease of hiring. TODO: more



[^1]: It's possible that business logic may be executed here, if the SPA is built to do server-side rendering. In that case, the controller might execute an internal build tool that makes the SPA effectivly "pre-bake" itself so that the first response can render fully formed, and skip the first round of URL parsing and data fetch requests. This technique can greatly increase the speed at which a SPA executes its first paint of DOM, but as you can imagine you now effectivly have 3 applications to manage (web app/api, SPA app, and server-side pre-render SPA app) because pre-rendering can have its own quirks and needs that a completly browser-side SPA does not have.
[^2]: This is not _entirely_ truthful. It depends entirely on how you decide to serve your API. Most SPAs do not live on an island. They still depend on a server somewhere handling data storage, doing authentication, etc. It's more accurate to say that your *web app server* has done its job, it has served your web app. Your API server still has a lot to do. [^3]
[^3]: If you say serverless, stfu. There's still a server out there, and you still have to cut code, but now you're tied to a 3rd party for critical infrastructure. Choose wisely. 
[^4]: You can also introduce massive amounts of friction between your new disparate teams. Be careful when splitting teams along specializations, it is rarely the clear-cut win one would assume it would be. Separate teams develop seperate cadences, seperate processes, separate *cultures* even. This is a decision that carries hidden costs that can be very hard see. Did Marcus leave because he was truly just "ready to do something different", or did the team split from last year decrease his ability to learn, introduce friction that made working less comfortable, making him feel like it was time to go? Can you measure this loss over time? Don't forget Conway's law, it is real. https://en.wikipedia.org/wiki/Conway%27s_law
[^5]: If you're lucky enough, /s, you might be used to calling this a "spotlet". https://www.quora.com/How-is-JavaScript-used-within-the-Spotify-desktop-application-Is-it-packaged-up-and-run-locally-only-retrieving-the-assets-as-and-when-needed-What-JavaScript-VM-is-used
[^6]: Or Canvas.
[^7]: If you think the FAANGs doesn't have this you're a goddamn fool. If the possability that this functionality could exist surprises you, uh, you should get off the internet. Like, forever. 
[^8]: A great example of how silod teams can subtily increase the complexity of an organization. Sarah could have fixed that slow search in two weeks if let loose, but she was on the wrong team, doing some other thing, and heard about the ES project too late for her to make the pitch. Now she's shaking her head at the sillyness of it all, and polishing up her resume. But hey, at least you have ES now! You're *ready for the future*[^9]
[^9]: You're not, but it certainly feels good to imagine you are. By the time the future comes around the problem is different, and so is the solution. But yay for good feels now!