# You Scaled Your Application... But Where Does the Data Go?

Imagine you've built a social media platform.

Initially, only a few friends and early users are using your platform.

People create profiles, share posts, follow other users, and interact through likes and comments.

At this stage, traffic is low.

A single application server connected to a single database node is more than enough.

```text
Users
  |
  ▼
Application
  |
  ▼
Database Node
```

The architecture is simple.

The application processes user requests, while the database node stores all important information:

- User profiles
- Posts
- Comments
- Likes
- Followers

Everything works perfectly.

But then your platform starts growing.

More users join every day.

More posts are created.

More interactions happen.

Soon, a single machine is no longer enough to handle the growing workload.

> **Note:** To keep this article focused on database architectures, we'll skip the details of application-layer scaling. In practice, multiple application nodes are added to handle more parallel requests, but the harder problem is scaling the data behind them.

Now the system looks like this:

```text
              Users
                |
                ▼
          Load Balancer
        /      |       \
       ▼       ▼        ▼
    Node1    Node2    Node3
                 |
                 ▼
           Database Node
```

Multiple application nodes can now process requests in parallel.

But every request still depends on the same database node.

The database node has become the bottleneck.

The engineering team first asks:

> **Can we make this database node more powerful?**

This approach is called **vertical scaling** (or **scaling up**).

We upgrade the existing machine by adding:

- More CPU to process more queries.
- More memory to cache frequently accessed data.
- Faster storage to improve I/O performance.

For some time, this works.

But eventually every machine reaches its physical limits.

A single database node becomes:

- Expensive to upgrade further.
- Difficult to scale beyond a certain point.
- A single point of failure.

The team now asks a different question.

> **Instead of making one database node bigger, can we use multiple database nodes?**

That immediately raises another question.

> **If we have multiple database nodes, how should they store and access data?**

Should every database node access the same storage?

Or should every database node own its own storage?

These two approaches lead to two fundamentally different architectures:

- **Shared Disk**
- **Shared Nothing**

---

# Shared Disk: What If Multiple Database Nodes Shared the Same Storage?

The first approach the engineering team can explores is simple.

> **What if we add more database nodes but keep the data in one shared place?**

This is known as a **Shared Disk Architecture**.

Instead of relying on one database node, we deploy multiple database nodes.

Each database node has its own:

- CPU
- Memory
- Query processing capability

But instead of having its own storage, every database node connects to the same shared storage.

```text
          Application Nodes
                |
                ▼
      ┌─────────────────┐
      │                 │
   DB Node 1         DB Node 2
 (CPU/RAM)          (CPU/RAM)
      │                 │
      └────────┬────────┘
               │
               ▼
         Shared Storage
```

Now the workload can be distributed across multiple database nodes.

At first glance, this looks like the perfect solution.

We don't need to think about:

- Which node owns a user's data.
- How to partition the data.
- How to move data between nodes.

Every database node can access every piece of data.

And this is a perfectly valid architecture.

Many systems successfully use Shared Disk because of its simplicity.

Since every database node sees the same data, development and operations remain relatively straightforward. There is no need to design complex partitioning strategies or decide where each user's data should live.

For many workloads, this simplicity is a huge advantage.

But sharing data comes with a hidden challenge.

Imagine two database nodes receive requests to update the same user's profile at exactly the same time.

```text
DB Node 1
Update email → user@example.com

DB Node 2
Update email → new@example.com
```

Both database nodes are trying to modify the same piece of data.

Without coordination, one update could overwrite the other, leading to inconsistent results.

To prevent this, the database relies on mechanisms such as:

- Locks
- Concurrency control
- Coordination protocols

These mechanisms ensure that only one node modifies a piece of data at a time, preserving consistency.

So far, everything still looks good.

In fact, this is exactly why Shared Disk works well for many systems.

But your social media platform continues growing.

Instead of **2 database nodes**, you now have **20**.

A few months later, you have **100**.

Every new database node still connects to the same shared storage.

Every new database node can still access the same data.

As more database nodes are added, more of them begin competing for the same shared resources.

This leads to:

- More lock contention.
- More waiting for shared resources.
- More coordination to maintain consistency.
- More network communication with the shared storage.

Adding another database node still increases compute capacity.

But it also increases the amount of coordination required.

Eventually, the coordination overhead starts eating into the performance gains.

The engineering team realizes something important.

> **The problem is no longer processing queries.**

> **The problem is that every database node is trying to share the same data.**

So another question naturally emerges.

> **What if database nodes didn't have to share everything?**

> **What if each database node owned its own portion of the data instead?**

This idea leads us to our second architecture.

# Shared Nothing

The engineering team starts thinking differently.

Instead of asking,

> **How can multiple database nodes share the same data?**

they ask,

> **Can we design the system so that database nodes don't have to share most of the data at all?**

This idea leads to the **Shared Nothing Architecture**.

Unlike Shared Disk, every database node now owns its own resources:

- CPU
- Memory
- Storage

Instead of sharing one storage system, each database node stores a portion of the data.

```text
          Application Nodes
                |
                ▼
      ┌───────────────────────┐
      │                       │
   DB Node 1              DB Node 2
 CPU / RAM / Disk       CPU / RAM / Disk
```

The first question that comes to mind is:

> **If every database node has its own storage, how do we decide where the data goes?**

The answer is **partitioning** (also called **sharding**).

The idea is simple.

Instead of storing every user's data on every database node, we divide the data into smaller pieces called **partitions**.

Each database node becomes responsible for only one partition.

For example, suppose we partition users based on their User ID.

```text
DB Node 1
Users: 1 – 1,000,000

DB Node 2
Users: 1,000,001 – 2,000,000
```

Now imagine User **245** updates their profile.

The application knows that User 245 belongs to **DB Node 1**, so the request is sent directly there.

```text
Application
      |
      ▼
DB Node 1
      |
      ▼
Disk 1
```

Notice something interesting.

Unlike Shared Disk, **only one database node owns this data**.

No other database node needs to access it.

No other database node competes to update it.

This immediately removes many of the coordination challenges we saw earlier.

There is:

- No shared storage for every database node to compete over.
- Far less lock contention across database nodes.
- Much less coordination to keep updates consistent.
- No central storage becoming a bottleneck as more database nodes are added.

This is the biggest advantage of Shared Nothing.

As traffic grows, we can continue adding more database nodes, each responsible for its own portion of the data.

The workload grows.

The data grows.

And the system grows with it.

But does this mean database nodes never communicate with each other?

**Not at all.**

Imagine User **245** is stored on **DB Node 1**.

User **1,500,321** is stored on **DB Node 2**.

Now User 245 decides to follow User 1,500,321.

This operation involves data that belongs to two different partitions.

The system now needs to coordinate across multiple database nodes.

This coordination happens over the network.

So Shared Nothing doesn't eliminate network communication.

It simply makes it much less frequent.

The goal is to partition the data so that **most requests can be handled by a single database node**, while only a small percentage require coordination across multiple nodes.

This is why choosing the right partitioning strategy is so important.

A good partitioning strategy keeps related data together, allowing the majority of requests to stay local and minimizing expensive cross-node communication.

Does this mean Shared Nothing is always the better choice?

Not necessarily.

Shared Nothing solves many of the scalability problems of Shared Disk, but it introduces new challenges of its own.

Engineers now need to think about:

- How should the data be partitioned?
- What happens when one partition grows much faster than the others?
- How do we rebalance data across database nodes?
- How do we handle operations that span multiple partitions?

In other words, we traded the coordination required by shared storage for the complexity of a distributed system.

---

## Final Thoughts

One thing I learned while studying distributed systems is that there is rarely a "best" architecture.

Shared Disk isn't obsolete.

Shared Nothing isn't automatically better.

Every architecture exists because it optimizes for a different set of trade-offs.

As engineers, our job isn't to memorize patterns—it is to understand **why** those patterns exist and **when** they should be applied.

If this article helped you think about scaling from that perspective, then it has achieved its goal.
