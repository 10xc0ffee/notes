# NOTES - Life Beyond Distributed Transactions: An Apostate's Opinion

## Problem Statement

Real-world application data grow fast. Those data will have to be repartitioned over time. Global serializability then becomes a problem. But user applications sometimes cannot get good performance by leveraging distributed transaction algorithms like 2PC (As mentioned by the Author in the year 2007).
So user application usually implements their own way to coordinate distributed procedures that fits their system.
This paper discussed a model/guideline for users to implement applications to deal with data scaling challenges, and for infra developers to build new frameworks.

## Assumption

- Application data will grow fast. But the types of data is limited.
- Application developers write code that is scale-agnostic.
- The infra provides multiple disjoint scopes of transaction serializability. (e.g single machine atomicity).
- For messaging, most applications like to use (or the infra usually provides) at-least-once delivery, but still at-most-once acceptance.
 - Exactly once like TCP protocol works fine for temporary non-persist messages in a short-live process level. But with the need for data durability, clearly more complicated to implement (it's like implementing a long-lived version of TCP connection. Refer to [RIFL](https://web.stanford.edu/~ouster/cgi-bin/papers/rifl.pdf) to see an implementation example).

## Pattern

With the above assumptions, the author provides the following design pattern of the scale-agnostic layer (a.k.a. application layer).

**Entity**

Addressable user data type. Atomicity happens at this level. Application is able to address an entity by a unique key. The author emphasizes the secondary index and primary index referenced data usually belong to different entities due to performance issues (coordination cost).

**Message**

The message is used to communicate across entities. When the update of entity A and the update of entity B are related (e.g. causality), there will be messages between A to B.
Messaging from entity A to entity B should be outside the transaction of either entity A or B. (For example, a user submits an order. The application updates the order table row A.
If that succeeds, then it needs to modify the balance table row B to charge the user. That may require a message that relates A to B. The messaging procedure should be outside transaction of A to prevent B receives the message before the transaction of A is committed).

**Activity**

The effect of a Message must be idempotent under the assumption of at-least-once-delivery. So the downstream entity needs to maintain some **state** to track those messages. That state is a part of an activity. Application is able to define more relationships (i.e. activities) between the two entities.
The author thinks if there is messaging between two entities, there should be an activity tracking these two entities. Under the assumption of limited user data types, the number of activities is also limited. So with this pattern, it doesn't mean application developers have to write a lot of code.
Another important state of the activity is the **uncertainty**. For example, we are going to charge the bank account, but before we do that we don't know how much money is in the account. That is uncertainty. The activity needs to save the state of uncertainty. The goal is to resolve the uncertainty. Considering if we are implementing a distributed transaction, normally the uncertainty is resolved along with the "locking", either pessimistic or optimistic. At the user application level, we track them as tentative operations, confirmations (like transaction committed), and cancellations (like transaction aborted).

**Workflow**
Application business logic will be composited by different messages and related activities (which are like building blocks).

### My Thoughts

In the year of 2007, it is just the time when NoSQL was rising up. This paper provides a design pattern to implement complex business logic above a NoSQL database with single row atomicity. Even though it is still complicated to implement such logic comparing to build it on a distributed RDMS. The application still needs to track/persist the activities and their states by itself, build a state machine to react to any activities' result, and handle all well-known failures in a distributed environment. I feel like it is tons of work.

Notice that many assumptions of this paper are probably not the case in current 2021. We already have many products that support exactly-once-semantic messaging, global transactions, and so on. That will basically make application development much easier.

But in certain areas, this pattern will still be useful.

The first use case I can think of is when the workflow is accessing multiple services/databases. Only accessing a single service has the atomicity. In some real-world business logics, a human can be an entity as well and can be involved in the workflow. So u have to break a global transaction into smaller transactions. I came across [7 Use Cases in 7 Minutes Each] (https://www.youtube.com/watch?v=sXGlQruUrWE), which talks about the real use cases by using AWS SWF. Basically, AWS SWF has many similar ideas and abstractions to help application developers focus on the business logic, without worrying about workload scaling (not data scaling) and distributed failures. SWF has the concept of activity worker and workflow worker, where the activity is like a simple step of business logic, the workflow is like a decision-maker/coordinator of different activities' execution. Application developers just need to break the business logic into activities and workflows. All other messaging, state durability and distributed failures are handled by SWF.