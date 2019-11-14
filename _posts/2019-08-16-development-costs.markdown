---
layout: post
title: "Ways to reduce Development Costs without Compromising On Quality"
date: 2019-08-16 21:30:00 +0600
description: "Advice on optimizing human resources and team management"
tags: [agile,software,development,project,management,mobile,application,business,finance,startup,digital,marketing,developer,lowering,costs,quality,human,resource]
comments: true
sharing: true
published: false
img: launch_001.jpg
---

Many startups and small companies today struggle with financing their software development business. They are forced to compete with IT giants in Silicon Valley for software developers, only to realize they cannot afford to pay 6-figure salaries (with equity and other perks!)

On the other hand, a talent that was swallowed up by a billion-dollar company, [complain](https://www.quora.com/How-is-life-after-leaving-Google-Why-do-people-decide-to-leave) about the stagnation of their skills due to lack use, which causes many of them to leave.

Where small businesses have to hire online-course graduates with zero hand-on experience, other companies are wasting precious human resources by hiring top engineers and giving them mediocre tasks that lack any challenge.

Frustrating, right?

But what if I tell you that you don't always need to hire top talent, yet still can get the expertise you need?

If your IT company wants to build a decent working product with a limited budget, there are three things you must do to succeed:

1. Change the workflow by assigning engineers the tasks that fit their skills best
2. Hire developers of different skill levels appropriate for the project needs
3. Save money with mediocre employees and expert consultants

Let’s go over each one in detail. 

### 1. Assign developers the tasks that fit their skills best
>
* If a person is underqualified to do the task, he is more likely to screw it up.
* If a person is overqualified to do the task, he ends up wasting his valuable time (and therefore, wasting your money as well).
>

When a junior developer builds the entire app, it may look fine on the surface, but then ends up crashing intermittently. A senior developer, on the other hand, is likely to implement everything correctly, but most of his time will be wasted on menial tasks.

In many companies I worked with, the same issue came up repeatedly: the firm hires a small team of engineers (the best they can afford) and assigns them scopes of work based on seemingly independent *features* of the product (different screens of a mobile app or services on the backend).

What happens is that the engineers get to the implementation of their independent scopes and often miss the opportunity of reusing the code done by their colleagues, by repeating the same work twice. Moreover, every developer adds their own *flavor* to coding, so a sibling *feature* may end up having a different inner structure.

The intuitive approach leads to accumulation of the technical debt at a frightening rate; the code base becomes a legacy for everyone on the project and a nightmare for the new developers joining it.

Even though there are ways to mitigate this problem (various Agile techniques: stand-up meetings, sprint planning, code reviews, pair programming, etc.), they don't address the root of the problem - **improper scoping of work**.

Let's take an example. We have a mobile app that starts with a user sign-up screen. In the context of this article, we can call this entire screen a *feature*.

If we break this feature down into tasks, we'll have something like this:

* User Interface (UI) work:
   * Aligning and styling the elements in a visual interface builder (easy)
   * Writing code for reusing the UI elements in other screens (medium)
   * Building custom UI elements (medium-complex)
   * Binding UI with a programming entity in the code (easy)

* Business logic:
   * Designing the module's infrastructure for better reusability and testability (medium)
   * Networking and interaction with databases (complex)
   * Interaction with other screens and routing (complex)
   * Connecting the business logic with the UI (easy)

What would a manager do in this case? That’s right: assign the whole thing to a developer!

Now, do you remember the statement I made about an overqualified and underqualified person? If we assign this to a junior developer, two-thirds of the tasks could possibly be completed with problems and bugs.

You may already start getting the point I'm about to make.

Yes, the tasks within the *feature* are often interconnected to such a degree that two developers would be unable to work in parallel on the same feature.

However, with a proper architectural design, the app's entities can be isolated so that most of the aforementioned tasks become independent from each other. This means that you can have one senior developer working on the networking requests and databases, a medium level engineer implementing the business logic, and a junior aligning the UI.

In this case, the scope of work for each developer is defined not by a feature, but by the task complexity and the engineer's skill level.

In practice, quite a lot of "features" in the project usually have similar tasks, so a group of such tasks would be a much more optimal and clear objective for an engineer.

This approach has more benefits than you may think:

* Optimal utilization of the team's expertise
* Improved productivity: each developer can focus on a specific type of task instead of switching the context between different ones.
* The development process starts to resemble Ford's conveyor belt with developers moving through the tasks at a faster pace than they would with a traditional assembly.
* Better code awareness. Engineers aren't isolated from each other's work; it's actually quite the opposite: they need to work closely to connect separate pieces. The code review becomes a natural part of the development flow.
* Better code quality. Every developer has to put their best effort into designing the programming interfaces for better modularity and reusability.
* Improved code consistency. For a developer, it's much easier to follow the unified structure of the module he designed for similar tasks in the past.
* The estimated time of arrival (ETA) of the project becomes more precise every time a task is complete since you can see the average time spent on analogous tasks by this developer and the number of such tasks left in the pipeline.

### 2. Hire developers of different levels appropriate for the project needs

Now you have your features broken down into individual tasks. Each task has an assigned complexity and ETA.

If you follow the tactics of grouping the tasks by complexity and similarity, rather than by feature, you can easily estimate how many developers of each skill level you need to complete the project by the deadline.

For example, the project appeared to require 800 hours for easy tasks, 1300 hours for medium, and 400 for complex (2500 hours total).

Then if the deadline is three months from now, we'll need around 6 people to be able to complete it in time: 3 months * 4 weeks a month * 35 hours a week * 6 people = 2520 hours of developers' work.

You can choose to hire 6 seniors or 4 medium + 2 seniors, but the theoretical minimal team would be 2 juniors, 3 medium, and 1 senior developer, proportional to the number of hours for easy, medium, and complex tasks respectively.

If you read the legendary book "The Mythical Man-Month" by Frederick Brooks, you know how many pitfalls are hidden in parallel software development; so you should ideally strengthen the team by choosing to hire a slightly better-skilled team than the minimum required. As it is also pointed out in that book, adding more people can introduce an unnecessary communication overhead that can eventually have no effect or even slow down the development.

### 3. Save money with mediocre employees and expert consultants

For every project, it's a good idea to have a "surgeon" on your team who can perform the most critical work and track the overall technical condition of the project.

The problem is just that such people are a rare asset, and their demands are higher just because they are smart enough to realize what they're worth.

Another problem is that you probably don't have enough "hard" tasks to keep them busy for months.

A relatively efficient practice is hiring contractors for the role of that "surgeon." With a contractor, you agree on the *results* with clear acceptance criteria, as opposed to agreeing on the *process* with employees.

Not only does this help you save a lot on payroll, but also, if the person is 10 times more productive than a mediocre developer, he'll be motivated to complete the work faster without comparing himself to other employees. You benefit from this as well, since the project wins some extra time.

As Frederick Brooks recommends in the aforementioned book, one of the best ways to save on development is to hire developers only when the architecture of the project is complete. Otherwise, the engineers may sit with no work lined up.

What happens in practice though is that they don't wait (it just sounds silly), so the project kicks off from multiple points, making it harder to bring the project to a common denominator later on.

In conclusion, hiring a consultant to build the architecture and other critical pieces of the system is a great example of where you can lower the costs of development without compromising on the quality.

In case you need a *surgeon* for your team - you know who to contact.