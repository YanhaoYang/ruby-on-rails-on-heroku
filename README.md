# Performance of Ruby on Rails on Heroku

Running big monolithic Ruby on Rails applications on Heroku could be challenging,
especially when there are large amount of data. For example, if Rails needs to
load too many data from database to process a request, and render complex views
to generate a response, it may take too much time, and use too much memory. When
it takes more than 30 seconds, Heroku will close the HTTP connection and log an
`H12 - Request timeout` error. When Rails uses more memory than the quota, Heroku
will log a `R14 - Memory quota exceeded` error.

Here are some relating articles, and some personal opinions regarding those problems.

## Quick fixes
Some fixes do not not need much effort to put in place, but you can see the effect
immediately.

### Replace small dynos with larger ones

> I recommend that all Ruby webapps run **at least 3 processes per server or container**. This maximizes routing performance.
> ... ...
> Routing at higher layers (that is, at the load balancer or Heroku’s HTTP mesh) is far more difficult to do so efficiently, because the load balancer usually has no idea whether or not the servers its routing to are busy or not.
>
> -- <cite>[Configuring Puma, Unicorn and Passenger for Maximum Efficiency][1]</cite>

More about Heroku's router:

> A set of routers automatically routes HTTP requests from your app’s hostname(s) to your web dynos. The router uses a **random selection algorithm** to distribute traffic across your web dynos.
>
> -- <cite>[Heroku Runtime][4]</cite>

### Use a server killer

> To make these memory leaks manageable, GitLab comes with the unicorn-worker-killer gem. This gem monkey-patches the Unicorn workers to do a memory self-check after every 16 requests. If the memory of the Unicorn worker exceeds a pre-set limit then the worker process exits. The Unicorn master then automatically replaces the worker process.
>
> -- <cite>[How GitLab uses Unicorn and unicorn-worker-killer][2]</cite>

Available killers:

* [unicorn-worker-killer](https://github.com/kzk/unicorn-worker-killer)
* [Puma Worker Killer](https://github.com/schneems/puma_worker_killer)

## Using Ruby wisely

### Less Ruby code

Because Ruby is relatively a slower language, it makes sense to move heavy computations to other components of the system. For example, use database for grouping, sorting and other calculations.

### Avoid allocating

Once Ruby VM requests some memory form OS, it seldom returns the memory to OS.

> However, one also need to consider that `free()` does not always mean the decrease of the amount of memory in use. If it does not return memory to OS, the amount of memory in use of the process never decrease. And, depending on the implementation of `malloc()`, although doing `free()` it often **does not cause returning memory to OS**.
>
> -- <cite>[Ruby Hacking Guide][3]</cite>

Therefore, a Ruby on Rails application should load least data.

* Filter data in database
* Only `select` necessary columns
* Caching data in memory to avoid repeated allocating
* Use in-place modification when possible

Compared to Ruby profiling and GC tuning, those are just some low-hanging fruits, but they are effective enough for web applications under moderate load.

### Use right data structures

For example, the time complexity of `Array.include?` is O(n), while the time complexity of `Set.include?` is O(1).

### Avoid Ruby global method cache invalidation

> The overall performance drop for multiple threads that work and multiple that invalidate is around 23,8%. Which means that if you have many threads (lets say you use Puma), and you work heavily with OpenStruct (or invalidate method cache in a different way) in all of them, you could get up to almost 24% more switching to something else. Of course probably you will gain less, because every library has some impact on the performance, but in general, the less you invalidate method cache, the faster you go.
>
> -- <cite>[Ruby global method cache invalidation impact on a single and multithreaded applications][5]</cite>

And here is another list of [Things that clear Ruby's method cache][6], with code examples:

## Heroku

### Streaming data to avoid timeout errors

According to Heroku's document [Request Timeout][7], "the router will terminate the request if it
takes longer than 30 seconds to complete.” 30s should be enough for most requests,
however, some requests may need more time. For example, a request to export a
large amount of data. This can be done in an asynchronous way. But if it has to
be synchronous, Rails's [Streaming][8] could be helpful.

### Enable preboot for smooth deployment

If the application takes more than a few seconds to boot, deployment will be possible to create timeout errors.
It happens like this. During the application is booting, requests are held by Heroku router. When the application
is ready to accept requests, Heroku router sends all those requests to the application immediately. If the
server does not have the capacity to process those requests, some of them will result in timeout errors.

See [the official document](https://devcenter.heroku.com/articles/preboot).

## Heroku Postgres

### Logging slow queries

Set [`log_min_duration_statement`][9] to a positive value representing how many milliseconds the query has to run before it's logged.

### Creating proper indexes

Add indexes for slow queries, then verify their effectiveness with `explain`.

> The most powerful tool at our disposal for understanding and optimizing SQL queries is EXPLAIN ANALYZE.
>
> -- <cite>[Reading a Postgres EXPLAIN ANALYZE Query Plan][10]</cite>

If a query only needs some special types of data, [partial indexes][11] might be helpful.

### Automatic vacuuming
Because of [MVCC][12], dead tuples will accumulate over time, resulting in bloated tables and indexes.
Vacuuming can keep the database clean. You may need to tune relating configurations to reduce its
impact on database performance.

> Once vacuum has fallen behind it consumes more resources when it does run and it interferes with normal query operation. This can lead to a vicious cycle where database administrators mistakenly reconfigure the “resource hog” autovacuum to make it run less frequently or not at all. Autovacuum is not the enemy, and **turning it off is disastrous**.
>
> -- <cite>[Postgres Autovacuum is Not the Enemy][13]</cite>

### Table partitioning

Partitioning breaks large tables in a series of smaller tables, so that the database server can load
less data into memory to execute some queries. When PostgreSQL can keep all data in memory, all queries
would be faster. If there is not enough memory, PostgreSQL needs to load data from disk to memory,
slowing down the responses dramatically.

[pgslice](https://github.com/ankane/pgslice) is a great tool to automate the process.
However it can only work with tables using integer as primary key.

## Articles


### [How not to structure your database-backed web applications: a study of performance bugs in the wild](https://blog.acolyer.org/2018/06/28/how-_not_-to-structure-your-database-backed-web-applications-a-study-of-performance-bugs-in-the-wild/)

Performance problems in some popular open source Rails applications and the ways to fix them:

* ORM API Misuse
  - Inefficient computation (IC) (`exists?` vs `any?`)
  - Unnecessary Computation (UC) (a query inside a loop body, that could be computed once outside the loop body)
  - Inefficient data accessing (ID) (N+1 selects problem)
  - Unnecessary data retrieval (UD) (an application retrieves persistent data that it doesn’t then use)
  - Inefficient Rendering (IR) (calls link_to within a loop to generate an anchor)

* Database design issues
  - Missing Fields (MF) (repeatedly computes a value that it could just store)
  - Missing Database Indexes (MI)

* Application design issues
  - Content Display Trade-offs (DT) (returning all records instead of using pagination)
  - Application Functionality Trade-offs (FT) (a side information on a page that is actually quite expensive to compute)

Here is the [original paper](https://hyperloop-rails.github.io/220-HowNotStructure.pdf).


### [Memory Bloat and "Leaks"](https://gist.github.com/nateberkopec/56936904705da5a1fa8e6f74cb08c012)

> The default Linux malloc will not release memory back to the OS unless it is at the end of the Ruby heap.


### [42 performance tips for Ruby on Rails](https://www.mskog.com/posts/42-performance-tips-for-ruby-on-rails/)

Some tips for performance. My favorite ones are
[Measure twice, optimize once](https://www.mskog.com/posts/42-performance-tips-for-ruby-on-rails/#measure-twice-optimize-once)
and [80⁄20 rule](https://www.mskog.com/posts/42-performance-tips-for-ruby-on-rails/#80-20-rule).


### [IMPROVE YOUR RUBY APPLICATION'S MEMORY USAGE AND PERFORMANCE WITH JEMALLOC](https://www.levups.com/en/blog/2017/optimize-ruby-memory-usage-jemalloc-heroku-scalingo.html)

Using jemalloc on Heroku, which may "reduce your Ruby application’s memory usage and response time".


### [Taming Rails memory bloat](https://www.mikeperham.com/2018/04/25/taming-rails-memory-bloat/)

> Set MALLOC_ARENA_MAX=2 everywhere you start Sidekiq and enjoy your extra memory.


### [Benchmarking Ruby's Heap: malloc, tcmalloc, jemalloc](http://engineering.appfolio.com/appfolio-engineering/2018/2/1/benchmarking-rubys-heap-malloc-tcmalloc-jemalloc)

> It looks like jemalloc gives conservatively an 11% speedup for a big concurrent Rails app.


### [Malloc Can Double Multi-threaded Ruby Program Memory Usage](https://www.speedshop.co/2017/12/04/malloc-doubles-ruby-memory.html)

> Multithreaded Ruby programs may be consuming 2 to 4 times the amount of memory that they really need, due to fragmentation caused by per-thread memory arenas in malloc. To fix this, you can reduce the maximum number of arenas by setting the MALLOC_ARENA_MAX environment variable or by switching to an allocator with better performance, such as jemalloc.


### [Scaling a Ruby on Rails app on Heroku](https://scottbartell.com/2019/03/26/how-to-scale-ruby-on-rails-app-on-heroku/)

A great summary about scaling a Rails app on Heroku.


### [PostgreSQL at Scale: Database Schema Changes Without Downtime](https://medium.com/braintree-product-technology/postgresql-at-scale-database-schema-changes-without-downtime-20d3749ed680)

When there are large amount of data in the database, simple migration task like adding a `NOT NULL` column can bring servers down.
Learn how to make database schema changes without downtime from this article.

There is also a wonderful gem, [strong_migrations](https://github.com/ankane/strong_migrations).
Detailed solutions are provided in its README.


### [Optimizing Database Performance in Rails](https://blog.heroku.com/rails-database-optimization)

> Nearly all of my performance investigations start with identifying slow queries, or views that are running far more queries than are necessary.


* [3 ActiveRecord Mistakes That Slow Down Rails Apps: Count, Where and Present](https://www.speedshop.co/2019/01/10/three-activerecord-mistakes.html)

> Look for uses of present?, none?, any?, blank? and empty? on objects which may be ActiveRecord::Relations. Are you just going to load the entire array later if the relation is present? If so, add load to the call (e.g. @my_relation.load.any?)
> Be careful with your use of exists? - it ALWAYS executes a SQL query. Only use it in cases where that is appropriate - otherwise use present? or any other the other methods which use empty?
> Be extremely careful using where in instance methods on ActiveRecord objects - they break preloading and often cause N+1s when used in rendering collections.
> count always executes a SQL query - audit its use in your codebase, and determine if a size check would be more appropriate.


## Books

* [Ruby Performance Optimization: Why Ruby Is Slow, and How to Fix It](https://pragprog.com/book/adrpo/ruby-performance-optimization)


[1]:https://www.speedshop.co/2017/10/12/appserver.html
[2]:https://about.gitlab.com/2015/06/05/how-gitlab-uses-unicorn-and-unicorn-worker-killer/
[3]:https://ruby-hacking-guide.github.io/gc.html
[4]:https://www.heroku.com/platform/runtime
[5]:https://mensfeld.pl/2015/04/ruby-global-method-cache-invalidation-impact-on-a-single-and-multithreaded-applications/
[6]:https://github.com/charliesome/charlie.bz/blob/master/posts/things-that-clear-rubys-method-cache.md
[7]:https://devcenter.heroku.com/articles/request-timeout#long-polling-and-streaming-responses
[8]:http://api.rubyonrails.org/classes/ActionController/Streaming.html
[9]:https://www.postgresql.org/docs/current/static/runtime-config-logging.html
[10]:https://thoughtbot.com/blog/reading-an-explain-analyze-query-plan
[11]:https://www.postgresql.org/docs/current/static/indexes-partial.html
[12]:https://en.wikipedia.org/wiki/Multiversion_concurrency_control
[13]:https://www.citusdata.com/blog/2016/11/04/autovacuum-not-the-enemy/
