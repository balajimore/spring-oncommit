# Beware of slow transaction callbacks in Spring

# TL;DR

If your application is failing to obtain new database connection, restarting ActiveMQ broker may help. Interested?

# The problem with performance

Few months ago we experienced a production outage.
Many requests were failing with all too familiar:

	java.sql.SQLTransientConnectionException: HikariPool-1 - Connection is not available, request timed out after 30003ms.
		at com.zaxxer.hikari.pool.HikariPool.createTimeoutException(HikariPool.java:555) ~[HikariCP-2.4.7.jar:na]
		at com.zaxxer.hikari.pool.HikariPool.getConnection(HikariPool.java:188) ~[HikariCP-2.4.7.jar:na]
		at com.zaxxer.hikari.pool.HikariPool.getConnection(HikariPool.java:147) ~[HikariCP-2.4.7.jar:na]
		at com.zaxxer.hikari.HikariDataSource.getConnection(HikariDataSource.java:99) ~[HikariCP-2.4.7.jar:na]
		at org.springframework.jdbc.datasource.DataSourceTransactionManager.doBegin(DataSourceTransactionManager.java:211) ~[spring-jdbc-4.3.4.RELEASE.jar:4.3.4.RELEASE]
		at org.springframework.transaction.support.AbstractPlatformTransactionManager.getTransaction(AbstractPlatformTransactionManager.java:373) ~[spring-tx-4.3.4.RELEASE.jar:4.3.4.RELEASE]
		at org.springframework.transaction.interceptor.TransactionAspectSupport.createTransactionIfNecessary(TransactionAspectSupport.java:447) ~[spring-tx-4.3.4.RELEASE.jar:4.3.4.RELEASE]
		at org.springframework.transaction.interceptor.TransactionAspectSupport.invokeWithinTransaction(TransactionAspectSupport.java:277) ~[spring-tx-4.3.4.RELEASE.jar:4.3.4.RELEASE]
		at org.springframework.transaction.interceptor.TransactionInterceptor.invoke(TransactionInterceptor.java:96) ~[spring-tx-4.3.4.RELEASE.jar:4.3.4.RELEASE]

In order to fully understand what's going, let's first take a look at what Spring and JDBC connection pool is doing underneath.
Every time Spring encounters `@Transactional` method it wraps it with `TransactionInterceptor`.
This interceptor will indirectly ask `TransactionManager` for current transaction.
If there is none, `AbstractPlatformTransactionManager` attempts to create new transaction.
In case of JDBC, `DataSourceTransactionManager` will start new transaction by first obtaining new database connection.
In the end Spring asks configured `DataSource` (`HikariPool` in our case) for new `Connection`.
You can read all that from aforementioned stack trace, nothing new.

# Very slow query

So what is the reason of given exception?
We are using Hikari as an example but the description is valid for all pooling `DataSource` implementations I'm aware of.
Hikari looks at its internal pool of connections and tries to return idle `Connection` object.
If there are no idle connections and pool is not yet full, Hikari will seamlessly create new physical connection and return it.
However if pool is full but all connections are currently in use, Hikari is helpless.
It must wait hoping that another thread will return a `Connection` in the nearest future so that it can pass it to another client.
But after 30 seconds (configurable timeout) Hikari will time out and fail.

What can be the root cause of this exception?
Imagine your server is working really hard handling hundreds of requests, each requiring database connection for querying.
If all queries are fast they should return connections fairly quickly back to the pool so that other requests can reuse them.
Even under high load waiting time shouldn't be catastrophic.
Hikari failing after 30 seconds may mean that all connections were actually occupied for at least half a minute, which is pretty terrible!
In other words we have a system that holds all database connections forever (well, for tens of seconds) starving all other client threads.

Apparently we have a case of terribly slow database query, let's check out the database engine!
Depending on the RDBMS in use you'll have different tools.
In our case PostgreSQL reported that indeed our application has 10 open connections - maximum pool size.
But that doesn't mean anything - we are pooling connections so it's desirable that under moderate load all allowed connections are open.
Only when application is very idle connection pool may decide to close some connections.
But it should be done very conservatively because opening the physical connection back is quite expensive.

So we have all these connections open according to PostgreSQL, what kind of queries are they running?
Well, embarrassingly, all connections are idle and the last command was... `COMMIT`.
From the database perspective we have a bunch of open connections, all idle and ready to serve transactions.
From Spring's perspective all connections are occupied and we can't obtain more.
What's going on?
At this point we are pretty sure SQL is not the problem.

# Simulating the failure

We looked at the stack dump of the server and quickly found the problem.
Let's look at the simplified piece of code that turned out to be the culprit after analyzing the stack dump.
I wrote a sample application [available on GitHub](https://github.com/nurkiewicz/spring-oncommit) that exposes the same problem:

	@RestController
	open class Sample(
			private val jms: JmsOperations,
			private val jdbc: JdbcOperations) {

		@Transactional
		@RequestMapping(method = arrayOf(GET, POST), value = "/")
		open fun test(): String {
			TransactionSynchronizationManager.registerSynchronization(sendMessageAfterCommit())
			val result = jdbc.queryForObject("SELECT 2 + 2", Int::class.java)
			return "OK " + result
		}

		private fun sendMessageAfterCommit(): TransactionSynchronizationAdapter {
			return object : TransactionSynchronizationAdapter() {
				override fun afterCommit() {
					val result = "Hello " + Instant.now()
					jms.send("queue", { it.createTextMessage(result) })
				}
			}
		}

	}

It's in Kotlin, just for the sake of learning it.
The sample application does two things:
* very, very simple database query, just to prove that it's not the issue
* post-commit hook that sends a JMS message

# JMS? 

It's pretty obvious by now that this post-commit hook must be the problem, but why?
Let's start from the beginning.
It's quite typical that we want to perform a database transaction and send a JMS message only when transaction succeeds.
We can't simply put `jms.send()` as the last statement in transactional method for few reasons:

* `@Transactional` can be part of a larger transaction surrounding our method but we want to send a message when whole transaction is done
* More importantly, transaction can fail at commit whereas we already sent a JMS message

These remarks apply to all side effects that do not participate in transaction and you want to perform then after commit.
Of course it may happen that transaction commits but post-commit hook is not executed, so the semantics of `afterCommit()` callback are at-most-once.
But at least we are guaranteed that side effect doesn't happen if data is not persisted to database (yet).
It's a reasonable trade-off when distributed transactions are not an option - and they rarely are.

Such idiom can be found in many applications and is generally fine.
Imagine you are receiving a request, persisting something to database and sending an SMS to a client confirming the request has been processed.
Without post-commit hook you'll end up with SMS being sent but no data written to database in case of rollback.
Or even _funnier_, if you are automatically retrying a failed transaction you may send several SMSes without any data persisted.
So post-commit hooks are important<sup>1</sup>.
What happened then?
Before looking at the stack dump let's examine the metrics that Hikari exposes:

![25 threads](src/main/img/25threads.png)

Under moderately high load (25 concurrent requests simulated with `ab`) we can clearly see that pool of 10 connections is fully utilized.
However 15 threads (requests) are blocked waiting for database connection.
They may eventually get the connection or time out after 30 seconds.
It still seems like the problem is with some long running SQL query, but seriously, `2 + 2`?
No.

# The problem with ActiveMQ

It's about time to reveal the stack dump.
Most of the connections are stuck on Hikari, waiting for connection.
These are of no interest to us, it's just a symptom, not the cause.
Let's look at the 10 threads that actually hold the connection, what are they up to?

    "http-nio-9099-exec-2@6415" daemon prio=5 tid=0x28 nid=NA waiting
      java.lang.Thread.State: WAITING
          [...4 frames omitted...]
          at org.apache.activemq.transport.FutureResponse.getResult
          at o.a.a.transport.ResponseCorrelator.request
          at o.a.a.ActiveMQConnection.syncSendPacket
          at o.a.a.ActiveMQConnection.syncSendPacket
          at o.a.a.ActiveMQSession.syncSendPacket
          at o.a.a.ActiveMQMessageProducer.<init>
          at o.a.a.ActiveMQSession.createProducer
          [...5  frames omitted...]
          at org.springframework.jms.core.JmsTemplate.send
          at com.nurkiewicz.Sample$sendMessageAfterCommit$1.afterCommit
          at org.springframework.transaction.support.TransactionSynchronizationUtils.invokeAfterCommit
          at o.s.t.s.TransactionSynchronizationUtils.triggerAfterCommit
          at o.s.t.s.AbstractPlatformTransactionManager.triggerAfterCommit
          at o.s.t.s.AbstractPlatformTransactionManager.processCommit
          at o.s.t.s.AbstractPlatformTransactionManager.commit
          [...73 frames omitted...]

All of these connections are stuck on ActiveMQ client code.
That's unusual on its own, isn't sending a JMS message suppose to be fast and asynchronous?
Well, not really.
JMS specification defined certain guarantees, some of which we can control.
In many cases fire-and-forget semantics are insufficient.
What you really need is a confirmation from the broker that the message was received and persisted.
This means we must:
* create a physical connection to ActiveMQ (hopefully it's pooled just like JDBC connections)
* perform handshake, authorization, etc. (as above, pooling helps greatly)
* send a JMS message over the wire
* wait for confirmation from the broker, typically involving persistence on the broker side

All of these steps are synchronous and not free, by far.
Moreover ActiveMQ has several mechanism that can further slow down the producer (sender):
[Performance tuning](http://activemq.apache.org/performance-tuning.html), 
[Async Sends](http://activemq.apache.org/async-sends.html),
[What happens with a fast producer and slow consumer](http://activemq.apache.org/what-happens-with-a-fast-producer-and-slow-consumer.html).
 
# Post-commit hooks, really?

So we identified that substandard ActiveMQ performance on the producer side was slowing us down.
But how on earth does this impact the database connection pool?
At this point we restarted ActiveMQ brokers and situation came back to normal.
What was the reason of producers being so slow that day? - that's beyond the scope of this article.
We got some time to examine Spring framework's code.
How are post-commit hooks executed?
Here is a relevant part of the invaluable stack trace, cleaned up (read bottom-up):

	c.n.Sample$sendMessageAfterCommit$1.afterCommit()
	o.s.t.s.TransactionSynchronizationUtils.invokeAfterCommit()
	o.s.t.s.TransactionSynchronizationUtils.triggerAfterCommit()
	o.s.t.s.AbstractPlatformTransactionManager.triggerAfterCommit()
	o.s.t.s.AbstractPlatformTransactionManager.processCommit()
	o.s.t.s.AbstractPlatformTransactionManager.commit()
	o.s.t.i.TransactionAspectSupport.commitTransactionAfterReturning()

Here is how `AbstractPlatformTransactionManager.processCommit()` looks like, greatly simplified:

	private void processCommit(DefaultTransactionStatus status) throws TransactionException {
		try {
			prepareForCommit(status);
			triggerBeforeCommit(status);
			triggerBeforeCompletion(status);
			doCommit(status);
			triggerAfterCommit(status);
			triggerAfterCompletion(status);
		} finally {
			cleanupAfterCompletion(status);  //release connection here
		}
	}

I removed most of the error handling code to visualize the core problem.
Closing (in reality, releasing back to the pool) of the JDBC `Connection` happens very late in `cleanupAfterCompletion()`.
So in practice there is a gap between calling `doCommit()` (physically committing the transaction) and releasing the connection.
This time gap is negligible if post-commit and post-completion hooks are nonexistent or cheap.
But in our case the hook was interacting with ActiveMQ and on that particular day ActiveMQ producer was exceptionally slow.
This creates quite unusual situation when connection is idle, all work has been committed, but we still hold the connection for no apparent reason.
It's basically a temporary connection leak.

# Solution and summary

I'm far from claiming this is a bug in Spring framework (tested with `spring-tx` `4.3.7.RELEASE`), but I'd be happy to hear the reasoning behind this implementation.
Post commit hook cannot alter the transaction or connection in any way, so it's useless at this point, but we still hold onto it.
What are the solutions?
Obviously avoiding long-running or unpredictable/unsafe code in post-commit or post-completion hook is a good start.
But what if you really need to send JMS message, make RESTful call or do some other side effect?
I'd suggest offloading side-effect to a thread pool and performing this asynchronously.
Granted, this means your side effect is even more likely to get lost in case of machine failure.
But at least you are not threating the overall stability of the system.

If you absolutely need to ensure side effect happens when transaction commits, you need to re-architect your entire solution.
For example rather than sending message immediately, store a pending request in a database within the same transaction and process such requests later, with retry.
This however can mean at-least-once semantics.

<sup>1</sup> That's the reason behind post-commit hooks in [Clojure transactional memory](https://clojure.org/reference/refs)

