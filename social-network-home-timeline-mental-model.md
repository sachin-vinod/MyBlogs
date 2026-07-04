# How Social Networks Build Your Home Timeline: A Mental Model

## Scope and Assumptions

This article focuses on understanding the trade-off between **Fan-out on Write (Push)** and **Fan-out on Read (Pull)** from the perspective of workload and computational cost.

Real-world systems consider many additional factors such as latency, consistency, memory usage, storage cost, cache hit rates, ranking algorithms, and operational complexity. To keep the mental model simple, we'll intentionally simplify the problem by focusing on one important design question:

> **When should a timeline be materialized?**

---

# 1. Introduction (Why should you care?)

Every time you open X, Instagram, or Facebook, your home feed appears almost instantly. Behind that seemingly simple experience lies one of the most important design decisions in distributed systems.

Should the platform prepare your timeline before you open the app, or should it build the timeline only when you request it?

At first glance, one approach seems obviously better than the other. But after working through a few scenarios, I realized there is no universally correct answer.

---

# 2. High-Level View

```text
Alice creates a post.
        ↓
Her followers expect to see it in their home timelines.
        ↓
Question:
When should those timelines be updated?
```

---

# Building a Common Vocabulary

> **Note:** Before comparing different approaches, let's understand a few terms that we'll use throughout the article. These terms aren't just jargon—they describe the core concepts behind how social networks deliver your feed.

## 1. Home Timeline

### Definition

A Home Timeline is the personalized feed a user sees when they open a social media application. It contains posts from the accounts they follow, along with other content depending on the platform.

### Why do we need it?

The entire discussion in this article revolves around how and when this personalized feed should be generated.

---

## 2. Timeline Materialization

### Definition

Timeline Materialization means preparing or updating a user's home timeline before they request it.

### Why do we need it?

Materialization is one possible strategy for making timeline reads faster by doing the work in advance.

---

## 3. Fan-out on Write (Push)

### Definition

Whenever a user creates a post, the system immediately updates the timelines of all relevant followers.

### Why do we need it?

This approach performs the work during post creation, so users can later read their timelines quickly.

---

## 4. Fan-out on Read (Pull)

### Definition

When a user creates a post, the system stores it but does not immediately update followers' timelines. Instead, the timeline is generated only when a user requests it.

### Why do we need it?

This approach avoids unnecessary work by generating timelines only for users who actually request them.

---

# Why Isn't Pull Enough?

Now that we've explored both approaches, let's see when each one makes sense.

We'll evaluate a few realistic workloads and understand why the best choice depends on how the platform is used.

---

# Why Not Simply Generate the Timeline on Every Request (Pull)?

Suppose we're building a social network similar to X.

A straightforward approach would be to generate a user's home timeline whenever they open the app.

To build the timeline, the system would:

1. Find all the accounts the user follows.
2. Fetch recent posts from each of those accounts.
3. Merge all those posts.
4. Sort them by timestamp (or ranking score).
5. Return the most relevant posts.

This approach is known as **Fan-out on Read (Pull)** because the timeline is generated only when the user requests it.

## But there's a problem...

Imagine Alice opens the app and keeps it open.

A few seconds later, Bob—someone Alice follows—publishes a new post.

How does Alice's application know that her timeline has changed?

One simple solution is **polling**.

Instead of waiting for the server to notify the client, Alice's application periodically asks:

> **"Has my timeline changed since the last time I checked?"**

For example, it might send the same timeline request every **5 seconds** while Alice is online.

This ensures Alice eventually sees Bob's new post, but it introduces a new problem.

Most of those polling requests won't find any new posts, yet the server still has to rebuild Alice's timeline every time.

Now let's look at what happens across the entire platform.

Assume:

- Users create **500 million posts per day**.
- Around **10 million users** are online simultaneously.
- On average, each user follows **200 accounts**.

If every online user polls the server every **5 seconds**, the platform must execute roughly **2 million timeline-generation queries every second**.

Since an average user follows around **200 accounts**, those millions of timeline queries translate into **hundreds of millions of post lookups every second**.

Even though generating a single timeline isn't particularly expensive, repeating the same work millions of times—many of which return no new posts—quickly becomes one of the most expensive operations in the system.

---

# Materializing the Timeline

Instead of repeatedly generating the same timeline, we can shift some of that work to the moment a post is created.

Whenever a user publishes a post, the platform looks up their followers and inserts the post into each follower's precomputed timeline.

Later, when followers open the app, the platform simply serves the already materialized timeline instead of rebuilding it from scratch.

This approach is known as **Fan-out on Write (Push)**.

Although the platform now performs additional work whenever a post is published, it significantly reduces the amount of work required for every subsequent timeline request.

For the average user, this trade-off is often worthwhile because the cost of updating a few hundred timelines is much smaller than repeatedly generating millions of timelines across the platform.

---

# The Celebrity Problem

The reasoning above works well for the average user because publishing a post typically requires updating only a few hundred timelines.

However, social networks also have users with millions of followers.

For these accounts, the same optimization becomes much more expensive. A single post may require updating millions of timelines before anyone has even opened the app.

This raises an important question:

> **Will all of those timeline updates actually be used?**

The answer depends on how users interact with that post. Instead of assuming all celebrity posts behave the same way, let's look at two different workloads.

---

## Scenario 1: Celebrity with Low Engagement

Suppose a celebrity has **100 million followers**.

Publishing a post requires materializing **100 million timeline entries**.

However, before the post becomes irrelevant, followers collectively request their timelines only **2 million times**.

### Which approach is better?

In this case, **Fan-out on Read (Pull)** is the better choice.

Materializing **100 million timelines** so that they are read only **2 million times** means most of the work provides little value.

Instead, the platform can generate the timeline only for users who actually request it, avoiding millions of unnecessary writes.

---

## Scenario 2: Celebrity with High Engagement

Now imagine the same celebrity announces a highly anticipated product launch.

- The celebrity still has **100 million followers**.
- Materializing the timelines still requires **100 million timeline writes**.
- This time, followers collectively request their timelines **500 million times** while the post remains relevant.

### Which approach is better?

Now **Fan-out on Write (Push)** becomes attractive again.

Although the platform performs **100 million timeline writes**, those writes are reused across **500 million timeline reads**. Instead of repeatedly generating timelines, the work is performed once and reused many times.

The key observation is that the same celebrity may require different strategies depending on how the post is consumed.

---

# What Happens in the Real World?

Real social networks serve millions of users with very different workloads.

For most users, publishing a post means updating only a few hundred timelines. Since this fan-out is relatively small, many platforms use **Fan-out on Write** as the default strategy. It reduces repeated timeline-generation work and provides fast reads.

Celebrities, however, are an exception. Their posts can require updating millions of timelines, making the write itself a large distributed operation.

Rather than applying the same strategy to every account, many platforms optimize for the common case and treat celebrities differently. For example, they may:

- Use **Push** for regular users.
- Use **Pull** for some celebrity posts.
- Or adopt a hybrid approach, where the strategy depends on factors such as follower count, expected engagement, or historical traffic patterns.

The exact implementation varies across platforms, but the mental model remains the same:

> **There is no universally better approach. The right choice depends on the workload and whether the cost of materializing timelines is justified by future reads.**

## References

This article was inspired by and based on concepts discussed in:

- Martin Kleppmann, *Designing Data-Intensive Applications (2nd Edition)*, Chapter 2.