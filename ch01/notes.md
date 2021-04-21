# Chapter 1: Reliable, Scalable, and Maintainable applications
- Introduction
    - Modern applcations are data intensive, as opposed to compute intensive
    - Standard building blocks used to create data intensive applications
        - database: stores data so it can be used again later
        - cache: remember the results of an expensive operation, so they can be used again later
        - search indexes: allow users to efficiently search for and index data
        - batch processing: periodically perform some compute intensive operation
- Purposes of the book:
    1. explore principles and practicalities of data intensive applications
    2. explore what different tools have in common, what distinguishes them, and how they achieve their characteristics
- Purpose of the chapter:
    1. define what we are trying to achieve when creating a data intensive application
- Three concerns that should be addressed when designing a data intensive application:
    1. Reliability - Does the system perform the task it was designed for, even under adverse conditions?
    2. Scalability - Does the application continue to perform well under an increased load?
    3. Maintainability - Can new engineers work on the system productively?
## Reliability
- An application "works correctly" if:
    1. The application is functionally correct; it behaves as a user would expect.
    2. The application can tolerate user errors or unexpected use cases.
    3. The application performs well for the required use case under the expected load and data volume.
    4. The application is secure. It forbids unauthorized access and abuse.
- An application is "reliable" if it continues to "work correctly", even when faults occur.
- A *fault* is anything that can go wrong in an application.
- An engineering team needs to define what kinds of faults their application is designed to tolerate.
- **faults vs. failures**:
    - a fault is when a single component of an application stops working as expected
    - a *failure* is when the system as a whole stops providing the expected service
- design concept: create mechanisms that prevent faults from causing failures
    - argument for approach: it is impossible to reduce the probability of a fault to zero
    - counter-intuitive approach to designing fault-tolerant applications: deliberately introduce faults
        - this ensures that there is appropriate error handling to prevent faults from becoming more widespread system failures
        - [netflix chaos monkey](https://github.com/Netflix/chaosmonkey) - randomly terminates components of a system in production to encourage engineers to build fault-tolerant applications
- there are some cases where preventing faults is preferable to tolerating them
    - ex: application security
- three types of faults
    1. hardware faults
    2. software errors
    3. human errors
### Hardware faults
    - hard disks crash, RAM becomes faulty, blackout in power grid
    - with a large datacenter, these kinds of hardware errors are increasingly prevalent
    - with cloud-based applications, it is increasingly common for hardware to become unavailable
    - solutions:
        - redundant hardware
        - designing fault-tolerant software
### Software errors
    - bugs that cause instances to crash when given bad input
    - runaway process that uses up some shared resource like CPU time, memory, disk space, or network bandwidth
    - another service that the system depends on becomes slow, unresponsive, or starts returning corrupted responses
    - cascading failures involving multiple services
    - solutions:
        - carefully thinking through the assumptions and interactions between components of the system
        - thorough testing
        - process isolation
        - regularly test that functional requirements and SLAs are met in production and raise an alert if they are not
### Human errors
    - configuration errors are the leading cause of outages
    - methods for reducing human error:
        - good system design
            - well designed abstractions, apis, and interfaces that are as simple as needed
        - decouple places where people make mistakes from places where outages are caused
            - provide sandbox environments for people to explore and experiment in
        - thorough testing
        - make it easy to recover from human errors
        - Use *telemetry*: make it easy to monitor performance and error rates
        - good management and training
### How important is reliability?
- reliability should still be considered when working on "non-critical" applications
- unreliable software can lead to lost customers and revenue
- If reliability is sacrificed for some other reason, it should be a conscious decision, and the project stakeholders should be made aware of the consequences.
## Scalability
- Scalability refers to an application's ability to cope with increased load
- It is meaningless to say an application is "scalable" or "not scalable". More context is required to describe **how** load is measured and how the engineers will adapt the system in response to increased load.
    - Need some quantification of load and another quantification of performance
### Describing load
- **load parameter**: a metric used to quantify how much stress the system is under
- choice of important load parameters depends on the architecture of a particular appliation
- common scenarios and appropriate load parameters:
    - web server: requests per second
    - database: ratio of reads to writes
    - chat room: number of simultaneous active users
    - cache: hit rate
### Case study: Twitter
- two main operations of twitter:
    1. post a tweet
        - 4.6k requests per second on average
        - 12k requests per second at peak usage
    2. read a user's timeline
        - 300k requests per second
- primary scaling challenge for twitter is fan-out: every post that a user makes needs to be published to all of their followers
- two approaches Twitter has used to deal with Fan Out
#### Approach 1: global collection of tweets
- maintain a central collection of tweets
- executes a query every time a user reads their home timeline to look up new tweets

#### Approach 2: cache each user's home timeline
- maintain a cache for each user's home timeline
- when a user posts a tweet, update each of their followers' caches

#### Comparison of the two approaches
- Approach 1 scales with the read rate, while approach 2 scales with the post rate
- Approach 2 is more scalable for Twitter's particular case, because the post rate is two orders of magnitude lower than the read rate.
- Approach 2 does not perform well for users with a huge number of followers. Twitter is transistioning to a multi-faceted approach that
    uses approach 1 for "celebrities" with many users, and defaults to approach 2 otherwise.

### Describing performance
- two questions to address:
    - when you increase a load parameter and leave the system resources unchanged, how is performance affected?
    - when you increase a load parameter, how do you need to change the system to keep performance the same?
- how to quantify performance depends on the type of application:
    1. for offline systems, like Hadoop batch processing jobs, *throughput*, or number of records processessed per second, is the metric of interest.
    2. For online systems, like web servers, the *response time* is a metric of interest
        - response time: time between a user sending a request and receiving a response
- These metrics are better thought of as random variables rather than a single number.
- Typically percentiles are reported for the metric. The average alone is usually not descriptive enough.
- High percentiles, or *tail latencies* are important because they quantify the experience of users in worst-case scenarios. These are the
users most likely to get frustrated by slow response times and leave the service.
    - Amazon study: a 100ms slowdown reduces sales by 1 percent
- reducing high percentiles has diminishing returns. It becomes harder to address them at higher percentiles because random errors have an increasingly large impact.
- Queueing delays often account for a large part of response times at high percentiles.
    - A server can only process a small number of tasks in parallel, so it is easy for a single slow request to hold up a line.
    - Important to measure response time from the client's perspective, because even though a single request may be processed quickly once it reaches the server, it may have spent a long time waiting in a queue before it even gets to the server.
- If a single user request needs to make several requests to other services, the slowest service dominates the overall response time.
    - this is called *tail latency amplification*
- Implementations of tracking percentiles for monitoring:
    - keep a rolling window of response times over the past 10 minutes
    - every minute, plot the median and various percentiles of the response times on a graph
    - naive implementation: maintain a sorted list of response times
    - approximation algorithms:
        - t-digest
        - HdrHistogram
    - the correct way to aggregate response time data from several machines is to add histograms

### Approaches for coping with load
- often need to redesign system architechture with incremental load increases
- **scaling up**: get a more powerful machine
- **scaling out**: distribute the load over many small machines
- **elasticity**: an *elastic* system automatically scales up or down to address changes in load. Elastic systems are good when the load varies unpredictably, but add to system complexity. The alternative is to scale manually.
- Distributed architectures typically add complexity. Conventional wisdom states that you should scale up until it is no longer cost effective to do so, then scale out.
- Effectively scaling a system requires a thorough understanding of which operations will be the most common and which will be rare.
- For an early stage startup, it is usually better to focus on developing features and then address scaling problems when the time comes.

## Maintainability





