
# Deep Dive

In this section, I will explain some of the main technical problems that I expected, how I solved them, and the trade-offs of these decisions. I will also explain some problems that can happen when the traffic become high.

## 1. Split Reads and Writes (CQRS)

In a job board, most of the traffic is reads. Many users will search for jobs, filter jobs and try to find matching jobs, while only a smaller number of users will apply for jobs or create new ones. If we put all this traffic on one PostgreSQL database, the database can get under heavy load. Also the indexes needed to make searching fast can affect the write performance.

To solve this, I used a **CQRS architecture**. PostgreSQL is mainly used for writes and important transactions such as job applications, while **Elasticsearch** is used for searching and matching. This allows us to scale the read side separately from the write side. The main trade-off here is **Eventual Consistency**. For example, if a user update his profile, the new information may not appear in search immediately. It can take some time until Elasticsearch receive the update.

This is acceptable for search because it is not a critical transaction. The important data is still in PostgreSQL and Elasticsearch is mainly for reading and searching.

## 2. Keeping Data in Sync - Outbox Pattern

Because we use PostgreSQL and Elasticsearch for different purposes, we need a way to keep the data between them in sync. Using distributed transactions between the two systems would make the system more complicated and also can be slow.

For this reason, we use the **Transactional Outbox Pattern**. When a user update his profile, we save the profile update and an event like `ProfileUpdated` in the same PostgreSQL transaction. A background worker then read the event from the Outbox table and send it to a message broker such as Kafka. After that, the search service can update Elasticsearch.

The good thing here is if Elasticsearch is down, we dont lose the event. It will stay in the Outbox until the worker can process it again. The trade-off is that now we have another table, worker and message broker to manage, so the system become more complicated.

## 3. Matching Engine - Elasticsearch

Matching candidates with jobs using normal SQL queries can become difficult when the amount of data becomes large. For example, we may need to compare many candidate skills with many job requirements, and the matching rules can become more complicated later.

Instead of doing all this work in PostgreSQL, we use **Elasticsearch** as the matching engine. Candidate profiles and jobs are stored as JSON documents, and Elasticsearch can use features such as `function_score` to calculate a matching score. For example, we can give more weight to **Must-Have** skills than **Nice-to-Have** skills.

The trade-off is that Elasticsearch use a lot of RAM and it adds another system that we need to manage. But for this type of search, the fast response time is worth it specially when the queries become more complex.

We also don't depend on Elasticsearch for the source of truth. PostgreSQL is still the main database and Elasticsearch is only used for the search side.

## 4. Preventing Duplicate Applications

Another problem is duplicate applications. For example, a user can click the Apply button two times because his internet is slow, or the frontend can retry the same request. Without protection, this can create two applications for the same job.

Also, it is important to understand that a **POST request is not idempotent by itself**. If we send the same POST request two times, normally it can create two different applications. So we need to add our own idempotency handling.

At the API level, we use an `Idempotency-Key` header. The client generates a unique key for the operation and sends it with the POST request. If the request is retried with the same key, the API knows that this is the same operation and can return the previous result instead of creating another application.

We also have protection at the database level. We add a **Composite Unique Index** on `candidate_id` and `job_id`. This is the final protection if something goes wrong in the API or if two requests somehow reach the database at almost the same time.

## 5. Skill Gap Calculation

When a user opens a job, we want to show him which skills he is missing. Doing a SQL `EXCEPT` query every time can add unnecessary load to the database, specially if many users are viewing jobs at the same time.

There is also a normalization problem. For example, `NodeJS` and `Node.js` can mean the same skill, but if we store them as text, the system can see them as different skills.

To solve this, skills are selected from a predefined list and each skill has an integer ID. The backend gets the candidate skills and the job skills and puts them into **HashSets**. We can then calculate the difference directly in application memory. HashSet lookup is O(1), so the overall operation is around O(N).

The trade-off is that this use a little more CPU from the application servers, but it reduce database work. Since a candidate normally has a small number of skills, the calculation is very fast and we don't need a complex solution for it.

## 6. Fan-out Problem - Notifications

A different problem happens when a popular employer publishes a new job. Imagine that the matching engine finds 10,000 candidates for the same job. If the API tries to send all these push notifications directly, the request can take too long and maybe timeout.

For this reason, we use background workers and a **Fan-out pattern**. The API publishes a `JobPublished` event and returns `202 Accepted` to the employer without waiting for all notifications to finish. A background worker receives the event, asks Elasticsearch for the matching candidates and divides the candidate IDs into smaller batches. These batches are then added to a notification queue, and other workers process them in parallel.

The trade-off is that the system become more complicated because now we need queues, workers, retries and a dead-letter queue for failed notifications. But the main API stays fast and don't need to wait for thousands of push notification requests.

If some notifications fail, they can be retried later instead of making the original job publish request fail.

## 7. Database Connection

PostgreSQL can handle a lot of traffic, but it does not like having too many open connections. If the traffic becomes very high and we have many application instances, each instance can create multiple database connections. Eventually PostgreSQL can reach its connection limit and new requests can start failing.

To reduce this problem, we use **PgBouncer** in front of PostgreSQL. PgBouncer manages the connections and allows many application connections to share a smaller number of real PostgreSQL connections. This means many requests can use the same database connections instead of opening a new one every time.

The trade-off is that PgBouncer is another component that we need to deploy and monitor. But it can protect PostgreSQL from connection exhaustion and make the system more stable when traffic become high.

It is not a solution for every database performance problem, but it help us avoid one common problem which is having too many connections at the same time.

