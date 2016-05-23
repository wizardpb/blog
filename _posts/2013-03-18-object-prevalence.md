---
layout: post
title:  "Object Prevalence: An In-Memory, No-Database Solution to Persistence"
date:   2013-03-18 11:23:32
categories: persistence in-memory-database scaling database event-sourcing
---
As machines become available with larger and larger amounts of main memory, in-memory databases are becoming more and 
more mainstream. One of the more interesting variants of an in-memory database is a pattern called Object Prevalence, 
and it has some characteristics that make it very applicable in today’s world.

Object Prevalence makes business objects persistent without a conventional database, so to start, let’s review the 
main characteristics of such a system.

- All data is held and operated on in main memory. In operation, there is no concept of reading and writing to a 
separate external store. Your whole business model is in memory, expressed in your favorite programming language, 
and operated on as such.
- Transactions and queries are instances of real objects, an expression of the Command pattern (see 
[http://www.oodesign.com/command-pattern.html][command-pat]). 
A prevalent implementation acts as an invoker of these, and will apply them to the business model.
- Transactions and queries are applied serially, so that they are isolated from each other, and can assume that the 
domain model is stable during their execution.
- Changes made by transaction objects are made persistent by serializing these objects to a journal, and replaying them 
on system startup. The journal is made durable, and writes atomic, so that it is immune to unexpected system crashes. 
To avoid continually having to replay from the start of time, periodic system snapshots are taken by serializing the 
whole of the business model to a snapshot file. Snapshots are taken at regular intervals, and allows replay of the 
journal only since the last snapshot.

Why would you want to do this?

- It’s fast. Very fast. Since everything runs in RAM, and the transactions are specifically crafted, an implementation 
can be orders of magnitude faster than a conventional database. Add SSD file storage for journals and snapshots, and 
these systems can give throughputs that are pretty much impossible with conventional databases.
-Memory and processing is getting very cheap. A multicore server with 1TB of main memory can be had for around $50K. If this 
pattern can save you that in staff costs (and it can), then … (UPDATE: As of mid-June 2013, a 1TB server from Dell is priced 
around 30-35K. Which makes this an even more compelling case).
- There is only one data model. I don’t know about you, but much of my development time on previous projects has been 
dealing with ORM issues – optimizing queries, dealing with non-optimal object models because of the needs of the database, 
and just handling two sets of tools. With Object Prevalence, all this is history. Everything is expressed in your 
favorite OO language, and you can use the full power of your debugging tools to fix every kind of problem in the system. 
To me, this is the biggest advantage of the pattern. Even with an ActiveRecord framework such as Rails or Grails, you will 
still have to tune the generated SQL and the object model to deal with the database. These can often be in conflict with the 
best design decisions for your domain model, which enforces a trade-off between the two – never an optimal situation. 
With Object Prevalence, you are completely free to design your domain model in a manner that is best for the application. 
This is no small gain for a complex application.

Generally, you will hold your business model in a single data structure. Its format is really up to you, but it should 
be serializable to a file. Call this the prevalent system. Then, this will be wrapped in the prevalence layer, which will 
provide mechanisms to take instances of your transaction and query objects and apply them to the model. When they execute, 
they will be passed a reference to this system, which they then transform in whatever way is appropriate. During this execution, 
the prevalence implementation guarantees that the model is stable – i.e. no other transaction or query will be running on it. 
It also guarantees that the transactions will be saved, and replayed in exactly the same order when the system is reloaded.

Querying the system is again done using explict objects, and as with Transactions, their execution defines what they will return. Queries cannot alter
the system state, and are interleaved with transaction executions. Since transations must be made durable before executing them,
and this is usually takes much longer than executing them, query execution is often overlapped with transaction logging.

If you are familiar with the Event Sourcing pattern, you will immediatley notice that Object Prevalance is an implementation of
this pattern. The events are the transaction objects themselves, and executing them defines the state transitions of the prevalent
system. Because the transactions are immutable, and the resulting state of an execution is simply a function of their value, the
state of the prevalent system at any time is completely defined by the execution order of the transaction objects. ACID properties of 
the system are built on this property:

- Atomicity is achieved because all transactions and queries are run serially, often using a single lock - this is possible because generally,
transaction execution is fast. The system moves 
through a series of defined changes, and can never appear to be in an inconsistent state when accessed via transactions and queries
- Consistency is defined by the transaction objects themselves, and is defined by the programmer. Each transaction defines what it’s 
consistency conditions are using explicit code that is part of the transaction. No longer are your consistency constraints 
spread over triggers, foreign key constraints, etc. This makes the system much more maintainable and understandable, as well as more robust
- Isolation is achieved, again, because queries are run serially with transactions. No queries can see inconsistent or partial results 
because no other changes are happening while they are running
- Durability is achieved by making sure the journal is written atomically – either the whole of the serialized transaction is written, 
or none of it is. The system will also ensure the writes are flushed to disk before executing the transaction. Snapshots are taken at regular 
intervals (again, interleaved with transaction execution), and allow journal replay from the last snapshot.
One very useful property of such a system is that it can be rewound – that is, it is possible move the system state back in time. This is very useful for debugging, as errors can be fixed, and the corrections replayed. It is also easy to replicate such systems. Since transactions are serializable, they can also be sent across a wire for reply on a replicated system. This keeps the two systems in sync.

It is also worth noting that most modern database installations are usually designed to run out of main memory anyway. This is done by 
using caches of all kinds, explicitly or implicitly. For example, Facebook has reported that about 90% of its queries never hit 
a disk at all. In this case, one is then accessing the main memory of a machine using a very inefficient interface – SQL. 
The overhead of this in execution and maintenance costs is high. With object prevalence, this is eliminated, with useful savings. 
The cost of a large main memory machine is less than the annual salary of a database administrator, so if prevalence can reduce staff 
count, the system will have already paid for itself – with the added benefit of faster development and better maintainability.

There are several open-source implementations of the pattern. The guys at prevayler.org were one of the implementers, and their 
Java open-source offering has some good examples in it. You can download the full source here: [http://prevayler.org][prevayler]. Another 
Java-based implementation is Space4j ([http://www.space4j.org][space4j]), and there are implementations in other languages also. A 
Ruby version exists at [http://github.com/ghostganz/madeleine][ghostganz].

There are some caveats to bear in mind when using the pattern. The most common mistake is making transactions 
non-deterministic i.e. not simple functions of their state. Since the persistent state is maintained by replaying transaction 
instances, they MUST produce exactly the same result on replay as on initial execution. This invariant can be violated in 
many and subtle ways, and is so common, that it’s often referred to as the ’Baptismal Problem’ – you have not been properly 
baptized in Object Prevalence until you have made it at least once!

Non-deterministic transactions can be created in many ways. Some common ones are

- Using files – you cannot guarantee the file will be the same, or even exist, when the transaction is replayed
- Creating dates or timestamps – they will not be the same on replay
- Allocating random numbers or UUIDs – again, they will vary on replay

Good implementations provide a notion of time that is invariant across transactions (The prevayler.org implementation does just that)

Secondly, transactions must not hold references to persistent business objects. Because they are serialized, these 
references will be different instances on replay. Instead, the domain model should have some kind of symbolic reference that 
can be used to refer to business objects – much as a primary key would do in a regular database. Transaction and query 
objects will then hold these, and look up objects using these references. In a web environment, this is not hard to do, 
as all requests will contain something that is (or can be used to derive) such a reference.

Thirdly, transactions are responsible for consistency, including cases in which they throw an exception. 
In other words – there is no rollback. This is often one of the most contentious issues when considering prevalence. 
A good prevalent implementation will handle exceptions on replay, but transactions must ensure consistency themselves.

In summary then, Object Prevalence will provide speed, ease development and maintenance, and can reduce staff costs – especially 
in applications with a complex object model. There is an excellent open-source Java implementation available, and an active 
mailing list.

Update: a comprehensive technical paper on the Prevalence pattern by Ralph Johnson and Klaus Wuestefeld can be found [here][paper]

[command-pat]: http://www.oodesign.com/command-pattern.html
[prevayler]:   http://prevayler.org
[space4j]:     http://www.space4j.org
[ghostganz]:   http://github.com/ghostganz/madeleine
[paper]:       http://hillside.net/sugarloafplop/papers/5.pdf