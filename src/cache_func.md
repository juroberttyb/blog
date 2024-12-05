# Func Caching

<p style="font-weight: bold">Oct 28, 2024 ~ Nov 21, 2024</p>

## Result

This is a post on how we optimize notification api response time from 

- <b>worst case over 100s to under 1s</b>
- <b>28% acceleration in average response time</b>

##### trace before <-> trace after
<img style="
  display: block;
  margin-left: auto;
  margin-right: auto;
  margin-top: 32px;
  margin-bottom: 32px;
  border-radius: 12px;
" src="./img/notif_final_trace.png"></img>

## Issue Description

A notification api is responsible for returning a user's notifications, such as new message, post response, new post..., but it takes over 100s to get the response in some cases and the overall response time is too long.

We find that the db query retrieving a list of notifications for a given user is the bottleneck for this api.

<img style="
  display: block;
  margin-left: auto;
  margin-right: auto;
  margin-top: 32px;
  margin-bottom: 32px;
  border-radius: 12px;
" src="./img/notification_db_query.png"></img>

## Our approaches

1) [Refactor Loop Queries](#refactor-loop-queries)
2) [Remove Worst-Case](#remove-worst-case)
3) [Parallelization](#parallelization)
4) [SQL Index](#sql-index)
5) [Min Heap](#min-heap)
6) [Func Caching](#func-caching)
7) [Future Work](#future-work)

## Func Caching

...
