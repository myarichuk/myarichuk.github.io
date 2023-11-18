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
In the testing realm, we encounter three major types of tests: unit testing, functional testing, and integration testing. Each plays a distinct role, and I'll discuss them briefly.

#### Unit Testing: The Building Blocks

Unit testing zeroes in on the 'small stuff'. It isolates and tests individual components or functions of your application. Picture it as examining each brick for flaws before building a wall. It ensures that every code segment performs exactly as intended, no more, no less.

Practically, unit tests often employ [mocks](https://www.geeksforgeeks.org/software-engineering-mock-introduction/) to *isolate* a code piece from its surrounding functionality.

#### Functional Testing: The User’s Perspective

Theoretically, functional testing involves evaluating your application from the user's perspective, ensuring it behaves as intended. This method doesn’t focus on the internals; rather, it treats the application as a black box and verifies if it meets specified requirements. Imagine testing whether the wall built from those bricks actually stands firm and fulfills its purpose.

In practice, this often involves mocking 'external' services, whether they're actual services in a microservice architecture or software modules not under test in a monolithic architecture.

#### Integration Testing: The Big Picture

Integration testing is where the collaborative magic happens. It assesses how different modules or services interact. The goal is to ensure that when code components interact, they do so as expected. Think of it as verifying that all the walls, floors, and ceilings in a building mesh well, forming a sturdy, functional structure.

For an integration test, you'd typically spin up a *real* system as part of the test, which may include multiple services and an **in-memory** database. To do this, you can use console functionalities or special language features, like those found in [.Net](https://learn.microsoft.com/en-us/aspnet/core/test/integration-tests?view=aspnetcore-7.0).

{% note info %}
Note that I have mentioned *in-memory* databases. While it is perfectly viable to set up tests with a persistent databases, it is much faster and requires less maintenance to use an in-memory mode of a database. Not all databases support such mode, but if they do, consider using it for integration tests.
{% endnote %}

#### Other Testing Types: Important But Situational

Beyond the big three of unit, functional, and integration testing, there are many other types. While these are crucial, they're akin to spices in a dish - used according to the need of the project. They’re situational, often tailored to specific project requirements.

* **Performance Testing:**  This ensures your system can withstand heavy loads without breaking. It’s also useful for identifying application breaking points and throughput.
* **Security Testing:** Consider this your app’s digital immune system check. In a world brimming with threats ranging from amateur hackers to advanced cybercriminals, security testing is not just important; it's imperative. However, its relevance may diminish for internal tools not intended for public release. This topic deserves its own blog post, or maybe three.
* **Automated Testing (automated UI testing):** A form of black-box testing where user activities are automated and UI interactions like button clicks are simulated. It's essential for repetitive tasks and a cornerstone of modern CI/CD pipelines.
* **Usability Testing:** Focuses on ensuring your app is intuitive and user-friendly. It's what differentiates a seamless user experience from a frustrating one.

#### And... Meaningful Tests?

I once joined a company boasting about their 3,000 tests and impressive test coverage. However, a closer look revealed a startling fact: their tests lacked [assertions](https://en.wikipedia.org/wiki/Test_assertion)! This rendered each test effectively meaningless. While this may be an extreme case, it underscores a vital point: not all tests are created equal.

{% note info %}
For those new to testing, here's the significance of assertions:

* Assertions in testing are akin to checkpoints in a race, confirming if you're on the right track. They're statements that validate whether a certain condition holds true. For instance, after executing a function, an assertion checks if the output aligns with your expectations.
* Without assertions, a test might execute your code but fail to verify its correctness. The test only confirms that the code runs without crashing, not that it produces accurate results
* Writing a test without an assertion is like cooking without tasting. You might follow the recipe (your code) to the letter, but without tasting (asserting), you can't be sure of the quality.
{% endnote %}

So, what makes a test both good and meaningful, beyond having proper assertions? First, let's consider the AAA pattern: *Arrange, Act, Assert*. This structure, along with its variant *given-when-then* from **Behavior-Driven Development (BDD)**, focuses on the end-user experience, translating user stories into tests.

{% note info %}
If you hear about BDD for the first time, no worries! I will take a look at BDD later in this post.
{% endnote %}

A key principle here is minimal scope focus. Essentially, tests are like a microscope, zooming in on a small area to reveal fine details. A good test narrows its focus to one specific code segment or functionality. This approach ensures clarity and conciseness in what's being tested. Unit or functional tests should isolate the part being tested for easier troubleshooting and maintenance. Integration tests, while not using mocks, should still focus on a single functionality aspect.

In complex systems, testing too many aspects simultaneously can obscure the cause if a test fails. Thus, focusing on a minimal scope is especially important.

Next, let's explore regression testing. Ever fixed a tough bug, feeling triumphant, only to encounter a new one taunting you? That's where regression testing, like a vigilant protector, steps in. It acts as a guardian, keeping past bugs and issues at bay, ensuring they don't resurface. After patching a bug or adjusting a feature, regression testing meticulously ensures no other part is adversely affected.

Then there's the 'happy path' testing, which replicates a flawless code execution path: no invalid inputs, transient errors, timeouts, or unhandled exceptions. While this is important, it often misses potential pitfalls.

* Ideally, tests should cover every code branch and possible exception.
* Practically, due to time constraints, it's common to start with the happy path. However, it's essential to identify and test code flows where significant issues might arise. Aiming for about 80% code coverage is a good metric, but it shouldn't be the sole focus. It's crucial to review the tests for their actual efficacy.

Lastly, let's touch on the balance between unit and integration tests. An abundance of unit tests can make even minor code changes ripple through your test base, increasing development costs. As your test suite grows, balancing unit tests with integration tests is advisable. Integration tests tend to perform 'black box' kind of testing, often reducing maintenance costs since we don't have to adjust tests for every single change we make in the system. Remember, developer time is a valuable resource!

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