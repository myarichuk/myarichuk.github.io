---
title: "'Production Ready' Non-Negotiable: Comprehensive Testing"
date: 2023-11-18T19:31:25+02:00
tags:
  - Software Engineering
  - Theory
  - DevOps
  - Production-Ready
categories:
  - Programming
  - Best Practices
top_img: production-ready-testing.jpg
cover: production-ready-testing.jpg

---

Let's delve into the most crucial, non-negotiable aspect of *production-ready* systems: Testing.

In my experience, opinions on testing vary widely, but unfortunately, some undervalue its importance or worse, take it for granted. I firmly believe that tests are indispensable; they act as a safety net, catching bugs before they can wreak havoc on a production system.
It may sound cliché, but this analogy accurately reflects reality.
So, let's begin with some taxonomy. Testing realm is a big one, but before anything else, it is worth mentioning the big three: unit testing, functional testing, and integration testing. Each plays a distinct role, and I'll discuss them briefly.

#### Unit Testing: The Building Blocks

Unit testing zeroes in on the 'small stuff'. It isolates and tests individual components or functions of your application. Picture it as examining each brick for flaws before building a wall. It ensures that every code segment performs exactly as intended, no more, no less.

Practically, unit tests often employ [mocks](https://www.geeksforgeeks.org/software-engineering-mock-introduction/) to *isolate* a code piece from its surrounding functionality and in this way, well written unit tests are designed to test *only* the code that is supposed to be tested.

#### Functional Testing: The User’s Perspective

At its core, functional testing is about looking at your application through the eyes of your users. It's not about digging into the code; instead, it's about asking, "Does this do what our users need?" Think of it as checking if the wall built from those bricks actually stands firm and does what it is supposed to.

We don't get tangled in the internals here. We treat the app as a black box - something where we're only concerned with what comes out, not how it’s done on the inside. This approach is key in making sure the app behaves as it's supposed to, based on what we've promised our users.

In practice, this often means mocking up 'external' services. Whether we're dealing with separate services in a microservice setup or just different modules in a single, monolithic application, we simulate parts not currently under test. This way, we can focus on how parts of our app interact and whether they live up to their promises in real-world scenarios.

In short, functional testing is all about those user stories and scenarios. It’s like saying, "Alright, the user wants to do X. Can they do it smoothly and exactly as the product owner defined application behavior?" It’s not just checking off features; it’s ensuring these features work in the way our users and product owners expect them to.

#### Integration Testing: The Big Picture

Integration testing is where we see if our app's different parts play nice together. It's about making sure that when various bits of our code meet, they interact just as we expect them to. Imagine ensuring that all the walls, floors, and ceilings in a building fit together perfectly, creating a solid, well-functioning structure.

In an integration test, it’s like we’re firing up a mini-version of our real system. This could include a bunch of services and usually an **in-memory** database to keep things fast and simple. To pull this off, we might need to use console commands to spin up services or use specific language perks, like the ones in [.Net](https://learn.microsoft.com/en-us/aspnet/core/test/integration-tests?view=aspnetcore-7.0).

{% note info %}
A quick heads-up: I've mentioned *in-memory* databases here. Sure, you could set up tests with regular, persistent databases, but using an in-memory database is like hitting the fast-forward button. It's quicker and cuts down on maintenance hassle. Keep in mind, though, not every database can do this in-memory trick. If yours can, it's definitely worth considering for your integration tests.
{% endnote %}

#### Other Testing Types: Important But Situational

Beyond the big three of unit, functional, and integration testing, there are many other types. While these are crucial, they're akin to spices in a dish - used according to the need of the project. They’re situational, often tailored to specific project requirements.

* **Performance Testing:**  This ensures your system can withstand heavy loads without breaking. It’s also useful for identifying application breaking points and throughput.
* **Security Testing:** Consider this your app’s digital immune system check. In a world brimming with threats ranging from amateur hackers to advanced cybercriminals, security testing is not just important; it's imperative. However, its relevance may diminish for internal tools not intended for public release. This topic deserves its own blog post, or maybe three.
* **Automated Testing (automated UI testing):** A form of black-box testing where user activities are automated and UI interactions like button clicks are simulated. It's essential for repetitive tasks and a cornerstone of modern CI/CD pipelines.
* **Usability Testing:** Focuses on ensuring your app is intuitive and user-friendly. It's what differentiates a seamless user experience from a frustrating one.

#### Crafting Tests That Truly Matter

I remember joining a company that bragged about their 3,000 tests and stellar test coverage. But here’s the kicker: overwhelming majority of those tests didn't have [assertions](https://en.wikipedia.org/wiki/Test_assertion)! They were like runners in a race without a finish line – going through the motions but not really achieving anything. This might be an extreme example, but it's a perfect reminder that quantity doesn't always mean quality in testing.

{% note info %}
In case you are new to this whole thing, here's the deal with assertions:

* Think of assertions in tests like checkpoints in a race. They're your reality check, confirming whether the code is performing as expected. For example, after a function runs, an assertion checks if the outcome is what you anticipated.
* A test without assertions might show that your code runs, but it doesn’t confirm it’s doing the right thing. It's like saying, "Hey, the engine's running!" without checking if the car is moving forward.
* Running a test sans assertion is akin to cooking a meal without tasting it. You might follow the recipe flawlessly, but you'll never know if it actually tastes good.
{% endnote %}

Before we talk about what makes a test meaningful, it is worth mentioning the design pattern we should write the tests with. Enter AAA pattern: Arrange, Act, Assert. Along with the given-when-then format from **Behavior-Driven Development (BDD)**, these structures ensure tests are not just about code, but about fulfilling user expectations.

For instance, consider a user story where a user logs into a system. The 'Arrange' part sets up the test environment, 'Act' involves the user entering their credentials, and 'Assert' checks if the system successfully logs them in. This structure ensures that each test is focused, clear, and relevant to user needs.

{% note info %}
New to BDD? No worries, I'll dive deeper into it later in the post.
{% endnote %}

One crucial thing in testing: focus on a minimal scope. Imagine tests as a microscope, zooming in to reveal the nitty-gritty of a specific code segment or functionality. Keeping tests narrow and focused helps in pinpointing issues precisely. Whether it’s a unit, functional, or integration test, zooming in on one aspect makes for easier troubleshooting and maintenance.

With complex systems, testing too many things at once can be like finding a needle in a haystack. By focusing narrowly, you make your life a lot easier when something goes awry.

Then there’s regression testing, our unsung hero. Ever squashed a bug and then another pops up, laughing in your face? Regression testing is your shield against this. After fixing something or tweaking a feature, it ensures the rest of your system isn’t thrown out of whack.

And let's talk about 'happy path' testing. It's like walking through a garden on a sunny day - everything's perfect, no hiccups. But, just like a garden isn't always sunny, our code isn’t always on its best behavior. Testing only the happy paths is like preparing for good weather without considering the chance of rain. Sure, starting with the happy path is common, but it's crucial to also venture down the 'unhappy paths'. These are the less-than-ideal scenarios – handling invalid inputs, dealing with timeouts, or managing unexpected errors.

By including these scenarios, we build a comprehensive test strategy that’s ready for anything. Think of it as having both sunglasses and an umbrella – being prepared for both the sunny days and the stormy ones. Aiming for around 80% code coverage is realistic, but remember, it’s not just about hitting numbers. It's more important to focus on how effective your tests are, ensuring they cover the full spectrum of what could go right and wrong.

Lastly, balancing unit and integration tests is key. Too many unit tests, and you might find yourself swamped every time there's a minor code change. As your test suite expands, mixing in integration tests can save you time and hassle. They’re more about the big picture and can be less maintenance-heavy, letting you focus on building great stuff rather than constantly tweaking tests. Remember, in the world of development, time is gold.

#### A Word Or Three About Testing Methodologies

Let's briefly explore a couple of important testing methodologies, particularly **Behavior-Driven Development (BDD)** and **Test-Driven Development (TDD)**, which have become ubiquitous in modern software engineering.

BDD and TDD are distinct yet complementary approaches. BDD, which evolved from the principles of TDD, is all about the user's interaction with the system. It involves starting with *user stories* ("As a user, I want...") and translating these narratives into specific tests. This approach ensures the software aligns perfectly with the user's needs and expectations. For example, in BDD, a test for a login feature would directly validate the user's ability to enter credentials and access the dashboard.

TDD, meanwhile, adopts a more code-centric approach. Here, the journey begins with writing a test for a specific function before even writing the function itself. The process follows a rhythmic cycle: write a test, watch it fail, develop the code to pass the test, and then refactor. Picture a TDD scenario for creating a calculator app, where the initial test for an 'add' function sets the stage for the actual coding and subsequent refinement.

Though different in their approach, both BDD and TDD significantly contribute to creating robust, user-focused software. They reinforce the importance of testing in not just catching bugs, but shaping the software's development from the ground up.

Apart from these, there are other methodologies like **Acceptance Test-Driven Development (ATDD)**, which extends TDD by involving client stakeholders in the testing process, and **Exploratory Testing**, where testers actively engage with the software to uncover unexpected behaviors or bugs without predefined tests.

## Series Roadmap

- 🔖 [**Introduction: Understanding Production-Ready Software**](https://www.graymatterdeveloper.com/2023/11/14/production-ready-intro/)
- 📈 [**Part 1: Structured Logging and Monitoring**](https://www.graymatterdeveloper.com/2023/11/14/production-ready-logging-monitoring/)
- ⏳ [**Part 2: Performance Metrics: Ensuring Reliability Under Pressure**](https://www.graymatterdeveloper.com/2023/11/15/production-ready-performance-metrics/)
- 🧰 [**Part 3: Memory Dumps: Deciphering the Black Box**](https://www.graymatterdeveloper.com/2023/11/15/production-ready-memory-dumps/)
- 🧪 [**Part 4: Comprehensive Testing: From Unit to Integration**](https://www.graymatterdeveloper.com/2023/11/18/production-ready-testing/) - *(You Are Here)*
- 🛡️ [**Part 5: Beyond the Basics: Security, Scalability, and Documentation**](#) *(Coming Soon)*

## What's Next?

Next, we will briefly explore the crucial yet often overlooked aspects of *production-ready* software, including security, scalability, and documentation.