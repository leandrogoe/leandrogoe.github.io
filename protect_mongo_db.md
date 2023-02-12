= Introduction =

Almost every SaaS platform has to manage some sort of background job processing. In some scenarios, background job spikes or a relatively small quantity of badly programmed jobs can affect the responsiveness of your system or even self inflict a denial of service. Here we will explore some techniques that can be used to prevent such scenarios when working with Sidekiq and MongoDB, but bear in mind that similar techniques could also be applied to other stacks.

= No tickets left! =

MongoDB is one of the leading databases engines on the NoSql space. WiredTiger is the default storage engine for MongoDB. As with any other decent database, WiredTiger supports concurrent writes and reads, and concurrency locks are handled at the document level for most operations. However, the level of concurrency that WiredTiger can manage is limited to 128 read requests and a same number of write requests by default. Those limits are typically informally referred to as the read a write tickets available in the system.

WiredTiger official documentation lacks an explanation of why this number was chosen or exactly why this mechanism was introduced, but unofficial sources suggest that the concurrency control is limited in this way in order to protect the database and the default limits were set high enough to ensure they would never be a bottleneck. So basically they are good thing and consultants don't recommend you to change them!

During operations it may happen that you see that read or write tickets go dangerously low, which means that you are putting too much strain on the database. If the number concurrent operations available does reach zero, all further operation requests start to queue, meaning that your database essentially stops responding until a ticket is freed.

As you might be guessing a relatively low number of very slow operations - 128 to be exact - or a greater number of slightly inefficient operations can easily deplete these tickets. Therefore it is crucial that you have some controls to prevent this ever happens, specially if you allow your background processing engine to autoscale.

= Sidekiq =

Sidekiq still remains an strong player in the realm of background processing engines that easily integrate with Ruby. When you work with Sidekiq, you need to decide how many Sidekiq processes you are going to spawn and how many threads each one of those will have. Furthermore, if you are working in the cloud, you may decide to autoscale the number of your Sidekiq processes based on load, so that you only pay what you need, what is a good thing.

You may be thinking, well we can just limit the maximum number of Sidekiq threads available some low number - a few tens of threads - and that would ensure we would never leave our database unresponsive. While that's mostly true, as your system grows, sooner or later you will need to scale that number of threads to much higher numbers. And that's feasible! If your jobs are not I/O intensive, you can easily scale Sidekiq to several thousands of threads without affecting the performance of your MongoDB cluster.

However, if you are in the scenario in which you have such a heavy background processing load, you may have noticed that any tiny mistake - like an unindexed query - can easily kill your database.

= Back off! =

If you know in advance that certain jobs can create great strain into the database, you can throttle them to prevent they consume too many WiredTiger tickets. There are already a few good libraries that integrate with Sidekiq and allow performing throttling for particular job classes. While manually throttling jobs is incredibly useful, sometimes we simply lack the information to add the proper throttling constraints to all the job classes in adavance. Moreover, a throttling configuration that works perfectly well in normal circumstances, it may prove too unrestrictive when different job types create load at the same time.

Ideally, we should aspire to a more intelligent control mechanism that can quickly react against the strain of the database and that prevents we ever reach a point in which the database has degraded to unacceptable levels.

The fact that we can easily use read a write tickets as a metric of the database strain is actually great, because it lets us build mechanisms using it as the main metric. We can easily get the number of tickets available through the `serverStatus` command:

```

```