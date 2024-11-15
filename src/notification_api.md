# It takes over 100s to get a notification list

<p style="font-weight: bold">Oct 28, 2024 ~ Nov ?, 2024</p>

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

## Issue Description

### Our apporaches

1) [Refactor Loop Queries](#refactor-loop-queries)
2) [Remove Worst-Case](#removes-worst-case)
3) [Parallelization](#parallelization)
4) [SQL Index](#sql-index)
5) [Read Replica](#read-replica)

### How we found the issue?

- [Alert] api latency over 2.5s for 5min

### What are the possible apporaches?

- db profiling, to maximize the performance of db query
  - how many times do we access disk?
  - parallel db connection, to speed up data retrieval
  - sql database upgrade, cpu amd mem capacity
  - data transmission
    - symbolic return, compress before returning from database
  - db index, to maximize the search/query speed of notification table
- parallel computation, to speed up business logic
  - synchronization, to ensure correctness
  - containerize, to allow pod horizontal scaling
- caching, to reduce number of api calls
  - inspector, to check if cached value is out-of-date
  - partial update, to avoid redundant update to up-to-date content
- smart sort, to optimize user experience, what they want to see?
- algorithm, is there a lower time complexity approach to the problem?
- notification microservice enhancement, the actual sender logic improvement
  - vertically, is it possible to make it faster?
  - horizontally, how less time a fcm message need to wait before sent
- system level, notification now is pull based, how about push based?
  - when new post arrived, we append the notif to the top of notif for audiences' post category's top notif?
  - graph database?

## Refactor Loop Queries

First, to find possible bottlenecks, we spread traces into notification api's bottleneck candidates, which led us to the discovery of 2 main bottlenecks and the following code
- the sql query to fetch a list of notifcations belonging to a user
- the aggregator function which aggregate the retreived data with related objects (eg: user, post, comments...)

##### legacy code which contains a loop of sql queries
```
for idx, user := range users {
    ...
    users[idx].Specialties, err = models.Specialty.GetByUser(gormDB, user.UserId, true)
    if err != nil {
      return nil, err
    }
    ...
}
```

Here we could see a db connection passed down a for loop, not a good sign.

By looking into traces, we could find the call at the bottom

<img style="
  display: block;
  margin-left: auto;
  margin-right: auto;
  margin-top: 32px;
  margin-bottom: 32px;
  border-radius: 12px;
" src="./img/notif_trace.png"></img>

Looking down the code, we findout that this loop create a new db connection each iteration, so we start to refactor this into a single query

```
// GetByUser ...
func (s *specialty) GetByUser(db *gorm.DB, userID string, latestOnly bool) ([]SpecialtyIntf, error) {
	...
	err := db.Model(s).
		Where("user_id = ?", userID).
		Order("ordinal ASC, updated_at DESC").
		Find(&specialties).Error
	...
}
```

One familiar with Golang may notice the legacy code uses Gorm to query database, since we as a team are moving to Sqlx, a new implementation would use Sqlx instead.

After converting a loop of sql queries to a single one, we now have the following implementation which retrieve a list of speciaties for a list of users.

##### aggregator/user.go
```
...
specialties, err := specialtyStore.GetUsersSpecialties(ctx, uids)
...
```

##### store/specialty.go
```
func (s *specialtyStore) GetUsersSpecialties(ctx context.Context, userIDs []string) ([]*models.Specialty, error) {
	
  query := `
		SELECT 
			id,
			application_id, 
			user_id, 
			specialty, 
			ordinal, 
			created_at, 
			updated_at, 
			deleted_at, 
			badge 
		FROM 
			public.specialties 
		WHERE 
			user_id = ANY($1) 
		ORDER BY 
			ordinal ASC, 
			updated_at DESC
	`

	...
	err := db.Select(&specialties, query, pq.StringArray(userIDs))
	...
}
```

Here we achieve the first significant acceleration! (a naive one I know)

We could now see the time consumed in ```aggregator.usersaggregator``` is down <b>from 357.4ms to 21.7ms</b>.

<img style="
  display: block;
  margin-left: auto;
  margin-right: auto;
  margin-top: 32px;
  margin-bottom: 32px;
  border-radius: 12px;
" src="./img/query_loop.png"></img>


## Removes Worst-Case

There is one more bottleneck which we could improve, as we could see <b>store.me.getmynotifs</b> is a heavy sql query.

##### notification list query
```
SELECT * FROM (
  SELECT *, row_number() OVER (PARTITION BY type, param_id ORDER BY created_at DESC)
  FROM notification
  WHERE is_removed=false
  AND (created_at>? OR type='0')
  AND (owner_id=? OR routing_type IN (?))
  AND creator_id!=?
  AND created_at<?
  AND type IN (?)
  AND (category IS NULL OR category IN (?))
  ORDER BY created_at DESC
) as r
WHERE (r.row_number=1 OR r.type = ?)
LIMIT ?
```

We could observe from the sql query plan that ```Bitmap heap scan``` is the most resource heavy part which most likely caused by the use of ```PARTITION BY type, param_id ORDER BY created_at DESC```. 

<img style="
  display: block;
  margin-left: auto;
  margin-right: auto;
  margin-top: 32px;
  margin-bottom: 32px;
  border-radius: 12px;
" src="./img/query_plan.png"></img>

From the original ```notification list query```, we could observe that the end goal is to retrieve notifications which are either 
   1) the latest notification of a ```type, param_id``` group, or
   2) notifications of a specific type
   
   for the first condition, we have found the use of ```DISTINCT ON (notification.type, param_id)``` only need to reach for the first row in each group instead of scanning the whole group, and give other benefits over the use of ```PARTITION BY type, param_id```.

  ```
  EXAMPLE QUERY
   ----------------------------------------------------------------------------------------------------
    SELECT DISTINCT ON (notification.type, param_id) *
    FROM notification
    ORDER BY notification.type, param_id, created_at DESC;
  ```

   ```
   QUERY PLAN for query using "DISTINCT ON (notification.type, param_id)"
    ----------------------------------------------------------------------------------------------------
      Unique  (cost=6101533.53..7917804.54 rows=54414 width=599)
        ->  Gather Merge  (cost=6101533.53..7843040.25 rows=14952858 width=599)
              Workers Planned: 2
              ->  Sort  (cost=6100533.50..6116109.40 rows=6230358 width=599)
                    Sort Key: type, param_id, created_at DESC
                    ->  Parallel Seq Scan on notification  (cost=0.00..414330.58 rows=6230358 width=59
   ``` 

   This change brings us

   -  removes remove O(n**2) worst case scenario to O(m*n), where m is the number of ```(notification.type, param_id)``` groups
   -  leads to 6~7% improvement in average response time
   -  <b>IMPORTANT</b> allow for further parallel optimization, see [Parallelized SQL](#parallelized-sql) section

   Note the above imrpovement is only possible since we parallelized requirements 3.1 and 3.2 into 2 sql queries.

##### before <-> after
<img style="
  display: block;
  margin-left: auto;
  margin-right: auto;
  margin-top: 32px;
  margin-bottom: 32px;
  border-radius: 12px;
" src="./img/trace_compare.png"></img>

With the use of ```DISTINCT ON (notification.type, param_id)```, we could now remove the use of ```row_number() OVER (PARTITION BY type, param_id ORDER BY created_at DESC)```, which <b>leads us to the removal of outer SELECT statement</b>.

   we could now form a new query (note requrement 3.2 is move to a standalone sql query)

   ```
			SELECT *
			FROM
			(
				SELECT 
					DISTINCT ON 
					(
						notification.type, 
						param_id
					) 
					...
				FROM 
					notification
				WHERE 
					...
				ORDER BY 
					notification.type, 
					param_id, 
					created_at DESC
			) AS r
			ORDER BY
				r.created_at DESC
			LIMIT 
				?
   ```

## Parallelization

Here we could see the code which implments parallelization for function ```notification aggregator```, but not in a very efficient way


##### notification aggregator
```
	um := make(map[string]models.UserSimple)
	uc := make(chan error)

	pm := make(map[string]models.Post)
	pc := make(chan error)

	rm := make(map[int64]models.ReadNotif)
	rc := make(chan error)

	go threadedReadMap(rc, rm, selfId)

	...

  go threadedPostMap(ctx, pc, pm, selfId, postIds)
  if err := <-pc; err != nil {
    return err
  }

  ...

  go threadedUserMap(ctx, uc, um, selfId, userIds)
  if err := <-uc; err != nil {
    return err
  }

  ...

	if err := <-rc; err != nil {
		return err
	}
```

Here we could notice 3 major issues

  1) go routines are spawned, but 2 out of 3 started waiting for their respective error channels, which leads to no speed gain
  2) instead of creating a ```threaded version``` of a function, go supports native threading in a more maintainable way to achieve parallelization.

      insteaf of threadedPostMap, we should do
      
      ##### better parallel
      ```
      go func(...) {
          returnValue, err := postMap(...)
          if err != nil {
              errCh <- err
          }
      }(...)
      ```

      and instead of waiting for each err respectively and sequentially, this is how we could do

      ##### improved err collection
      ```
      go func() {
          aggregateNotifGroup.Wait()
          close(errCh)
      }()

      if err, exist := <-errCh; exist && err != nil {
          if exist {
            logging.Errorw(ctx, "failed to get notification list", "err", err)
            return nil, err
          }
      }
      ```

      now we would receive a done signal to know the child routines run correctly, which achieves better parallelization.

  3) better error collection

      as mentioned above, with the inclusion of done channel we could wait after spawning multiple go routines.

      ##### parent go routine

      ```
      ctx, cancel := context.WithCancel(ctx)
      defer cancel()

      errCh := make(chan error)
      go func(errCh chan<- error) {
        ...
      }(errCh)
      ```

      ##### child go routines
    
      ```
      for ... {
        select {
        case <-ctx.Done():
          return
        default:
          ...
          if err != nil {
            logging.Errorw(ctx, ...)
            errCh <- err
          }
      }
      ...
      ```

      this patterns ensure we don't get zombie threads from parent routines failing before child routines.

  So, after the adoption of go routines, waitGroup, withCancel pattern, and err channel, we have better appoaches to the 3 issues and achieve a much better result. 

##### before <-> after
<img style="
  display: block;
  margin-left: auto;
  margin-right: auto;
  margin-top: 32px;
  margin-bottom: 32px;
  border-radius: 12px;
" src="./img/serious_parallel.png"></img>

  Now all precedures below ```aggregator.notification.notifaggregator``` are parallelized, but because it itself is also parallelized where it is called, its trace is very long, the real heavy logic part of it only take place after ```dev.store.me.getmynotifs``` is done.

## SQL Index

Now, the greatest bottleneck, ```dev.store.getmynotifs```

Let's see the query again, see what we could do

##### heaviest query
```
SELECT * FROM 
(
  SELECT 
    DISTINCT ON 
    (
      notification.type, 
      param_id
    ) 
    *
  FROM 
    notification
  WHERE 
    is_removed=false
    AND 
    (
      created_at>? 
      OR 
      type='0'
    )
    AND 
    created_at<?
    AND 
    (
      owner_id=? 
      OR 
      routing_type IN (?)
    )
    AND 
    creator_id!=?
    AND 
    type IN (?)
    AND 
    (
      category IS NULL 
      OR 
      category IN (?)
    )
  ORDER BY 
    notification.type, 
    param_id, 
    created_at DESC
) AS r
ORDER BY
  r.created_at DESC
LIMIT 
  ?
```

##### using index
<img style="
  display: block;
  margin-left: auto;
  margin-right: auto;
  margin-top: 32px;
  margin-bottom: 32px;
  border-radius: 12px;
" src="./img/query_plan_compare.png"></img>

Types of index we could consider
  
  - ...
  - ...

Now, since I am not a sql database expert, I leverage the help of one of our most important big brains... ChatGPT!

After some chat with him (her? it?), I eventually take two of his suggestions.

  1) ...
  2) ...

Since the main bottleneck for our sql query is for the getting notifications part not the superLike part and it is parallelized so not affecting overall response time, we would only add indexes for the former query.

Note adding indexes itself also incur extra execution time for writting to the table, so ideally we are not going to see lots of them if there is no clear improvement.

## Read Replica

Now, since this is a heavy sql query which does not involve any write operation, we could consider moving its work to a read only sql replica.
  
  - ...
  - ...
