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

### How we found the issue?

- [Alert] api latency over 2.5s for 5min

### What need to be done before taking actions?

- bottleneck identification
  - heavy use of tracing, to profile current logic flow
- note in this case, we are dealing with a lot of legacy code, so heavy testing would be required

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

### Iteration 0, replace a loop of sql queries into a single one

1) traces are spreaded into notification api's possible bottlenecks.
2) we discovered 2 main bottlenecks, one is the sql query to fetch notifcation data, the other is the aggregator which aggregate the retreived data with related objects.
3) so first, we find the legacy code below to be one of the bottlenecks

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

4) here we find a db connection passed down in a for loop, not a good sign
5) by looking into trace structure, we find the call at the bottom

<img style="
  display: block;
  margin-left: auto;
  margin-right: auto;
  margin-top: 32px;
  margin-bottom: 32px;
  border-radius: 12px;
" src="./img/notif_trace.png"></img>

6) by more investigation, we findout that this for loop indeed use a new db connection to fetch user each iteration, so we decide to refactor this into a single db query

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

7) one may notice the legacy code uses Gorm to query database, since we as a team are moving to Sqlx, a new implementation would use Sqlx instead.

8) after converting a loop of sql queries to a single one, we now have the following implementation which retrieve a list of speciaties for a list of users

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

9) then here we achieve the first significant acceleration, we could see that the time consumed in ```aggregator.usersaggregator``` is down <b>from 357.4ms to 21.7ms</b>.

<img style="
  display: block;
  margin-left: auto;
  margin-right: auto;
  margin-top: 32px;
  margin-bottom: 32px;
  border-radius: 12px;
" src="./img/query_loop.png"></img>


### Iteration 1, improve sql query which removes worst case scenario

1) From the above trace structure, there is one more bottleneck which need to be dealt with, which is <b>store.me.getmynotifs</b>, a heavy sql query.

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

2) As we could observe from the sql query plan, the ```Bitmap heap scan``` is the most resource heavy part which most likely caused by the use of ```PARTITION BY type, param_id ORDER BY created_at DESC```. 

<img style="
  display: block;
  margin-left: auto;
  margin-right: auto;
  margin-top: 32px;
  margin-bottom: 32px;
  border-radius: 12px;
" src="./img/query_plan.png"></img>

3) from the original ```notification list query```, we could observe that the end goal is to retrieve notifications which are either 
   1) the latest notification of a ```type, param_id``` group, or
   2) a notification with a certain type
   
   for the first condition, we have found the use of ```DISTINCT ON (notification.type, param_id)``` give better performance than the use of ```PARTITION BY type, param_id```.

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

   This change <b>remarably removes worst case scenario and leads to 5~6% improvement in response time on average</b> since the scan no longer need to walk through the whole group but only need to return the first one in each group with the use of ```ORDER BY notification.type, param_id, created_at DESC```.

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

4) with the use of ```DISTINCT ON (notification.type, param_id)```, we could now remove the use of ```row_number() OVER (PARTITION BY type, param_id ORDER BY created_at DESC)```, which <b>leads us to the removal of outer SELECT statement</b>.

   we could now form a new query

   ```
   ```

### Iteration 2, adding index after query optimization

