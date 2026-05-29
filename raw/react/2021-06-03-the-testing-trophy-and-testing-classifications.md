# The Testing Trophy and Testing Classifications

> Source: https://kentcdodds.com/blog/the-testing-trophy-and-testing-classifications
> Collected: 2026-05-29
> Published: 2021-06-03

Allow me to indulge in a little personal history. If you're unfamiliar with the testing trophy, here it is:

![Illustration of a trophy separated into 4 sections labeled from top to bottom: End to End, Integration, Unit, Static](https://res.cloudinary.com/kentcdodds-com/image/upload/f_auto,q_auto,dpr_2.0,w_1600/v1622744540/kentcdodds.com/blog/the-testing-trophy-and-testing-classifications/trophy_wx9aen.png)

I initially introduced this in a tweet with a quick drawing I made with Google Drive:

> "The Testing Trophy" 🏆 A general guide for the **return on investment** 🤑 of the different forms of testing with regards to testing JavaScript applications. - End to end w/ @Cypress_io ⚫️ - Integration & Unit w/ @fbjest 🃏 - Static w/ @flowtype 𝙁 and @geteslint ⬣
— Kent C. Dodds, February 6, 2018

I came up with this idea after publishing a blog post titled "Write tests. Not too many. Mostly integration." Which was my take on Guillermo Rauch's tweet from about a year earlier:

> Write tests. Not too many. Mostly integration.
— Guillermo Rauch, December 10, 2016

I can't speak for Guillermo, but I agreed so strongly with what he said because of my experience as a UI engineer and how I personally had come to understand the term "integration" in this context.

Especially at that time in my career, almost all the code I wrote either ran directly in a browser or was intended for a tool that would help me run code in a browser. So for me naturally the terms "unit", "integration", and "end-to-end" would be viewed through the lens of that experience. In fact, I added "static" to the trophy because in the world of JavaScript that's not a given like it is in the predominant languages when the testing pyramid was introduced.

The reason I explain this background is to help you understand the way the Testing Trophy is intended to be interpreted. I never considered whether it applied to microservices or even backend services at all. I considered my codebase in isolation and attempted to categorize the types of tests I could write within the confines of my own code ownership. I always thought of end-to-end tests as the place where you attempt to validate that things work without any (or more practically "as little as possible") mocking in place.

So that left me with categorizing tests on my own code into either "unit" or "integration". I consider a "unit" to be a single function, class, or object that contains logic. So here's how I decided to (loosely) categorize them:

- Unit tests are those which test units which either have no dependencies (collaborators) or which have those mocked for the test.
- Integration tests are those which test multiple units integrating with one another.

Eventually, I created Testing Library to encourage the kinds of testing practices that worked best for me. By my own definition, Testing Library can be used to test individual React components (unit tests), entire pages with HTTP requests mocked via MSW (integration tests), the full app with very few mocks (end-to-end tests), and even individual React hooks if necessary (lower level unit tests). And Testing Library is now the most popular and de facto standard testing library for React apps.

I expect some will reply to this blog post with: "Why did you have to make up your own definitions in the first place? Just use the ones that exist." So I'll respond before you ask: "Which of the two dozen different definitions would you like me to have chosen for my own definition?" In his post about test shapes, Martin Fowler approximates a quote of a "test expert" who was asked in the 1990s how they define "unit test":

> "in the first morning of my training course I cover 24 different definitions of unit test".

This is a sad state of affairs, and it's been that way since the 90s unfortunately. It is what it is. I had to choose something that made sense for me and as an educator, I had to choose something that would make the most sense for the people I'm teaching. Judging by the response from people who have implemented my recommendations, my decision was a good one.

When discussing whether you can prove that testing is effective, Tim Bray (in his article "Testing in the Twenties"), correctly says:

> let's not kid ourselves that our software-testing tenets constitute scientific knowledge.

I would say this applies to everything about testing–not just whether it's effective (it can be). Any attempt to come to a single definition for all these terms is a futile endeavor.

Justin Searls said it best when he tweeted:

> People love debating what percentage of which type of tests to write, but it's a distraction. Nearly zero teams write expressive tests that establish clear boundaries, run quickly & reliably, and only fail for useful reasons. Focus on that instead.
— Justin Searls, May 15, 2021

Classification is important so we can have conversations about this. It's unfortunate that you pretty much need to come to a consensus on how you define these terms before having a productive conversation. But ultimately it really doesn't matter. As Justin says, it's a distraction. Especially when so many codebases are living life on the edge without an automated way to have confidence their changes are safe to deploy.

## Conclusion

Anyway, hopefully this helps to clear things up a bit. To sum up: When trying to apply the testing trophy to your situation, think of it within the code of an individual codebase. It definitely has applicability in backends, but I've only considered it for monoliths not microservices or even serverless functions (and I agree with Tim, most of us should probably be writing monoliths if we can).

The testing trophy (when understood) has given me (and countless other) clarity on where to focus testing efforts. When properly interpreted, it helps me keep this critical principle in mind:

> The more your tests resemble the way your software is used, the more confidence they can give you.
— Kent C. Dodds, March 23, 2018

This is the guiding principle for Testing Library and it's how I think about every testing problem I face.

**Remember,** it's all about getting a good return on your investment where "return" is "confidence" and "investment" is "time." If we had unlimited time, then trying to classify things wouldn't be necessary, we'd just write tests forever! But we don't, so I hope this helps you when trying to decide where to put your efforts.
