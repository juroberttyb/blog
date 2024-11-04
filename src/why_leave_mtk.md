# Why

This is a thread on how we optimize notification api response time.

Long word short, we find that the db query retrieving a list of notifications for a given user is the bottleneck for this api.

<img style="
  display: block;
  margin-left: auto;
  margin-right: auto;
  margin-top: 32px;
  margin-bottom: 32px;
  border-radius: 12px;
" src="./img/notification_db_query.png"></img>

- how we found the issue
  - [Alert] api latency over 2.5s for 5min
- possible approaches
  - mock on previous outputs, to show requirements
  - parallel db connection, to speed up data retrieval
  - parallel computation, to speed up business logic
  - synchronization, to ensure correctness
  - caching, to reduce number of calls
  - inspector, to check if cached value is out-of-date
  - partial update, to avoid redundant update to up-to-date content
  - symbolic return, compress before returning from database
  - logging and tracing, to obtain deep profiling on current bottleneck
  - db profiling, to maximize the performance of db query
    - how many times do we access disk?
  - db index, to maximize the search/query speed of notification table
  - smart sort, to optimize user experience, what they want to see?
    - sql database upgrade, cpu amd mem capacity
  - algorithm, is there a lower time complexity approach to the problem?
  - system level, notification now is pull based, how about push based?
    - when new post arrived, we append the notif to the top of notif for audiences’ post category’s top notif?
    - graph database?
