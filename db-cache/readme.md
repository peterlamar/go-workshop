# db-cache

This is an example using redis and postgres in a db caching pattern. This uses
a cache (in the case redis) to as a first stop in getting data from a database.
This builds on the traditional relational database pattern of retrieving data
from the database with sql queries and adds a new layer of cache between the
application and the database


### DB Cache pattern design

backendapp (golang) <--> cache <--> postgres


* Case 1 - No data in cache
The application checks the cache if it has a sql query result. Since a cache may
be accessed using a [key/value](https://redis.io/topics/data-types-intro) pair,
the cache can be checked using a unique key (the key can be the entire sql query
or a string comprising of the table name and its arguments). If the key does not
exist in the cache, then the Sql query is run against the Postgres db and the
data is retrieved per the normal pattern. The data is then stored into the cache
for next time the query must be run using a unique key as discussed and the data
as the corresponding value to the key. The cache may be set with a timer that
[auto expires](https://redislabs.com/ebook/part-2-core-concepts/chapter-3-commands-in-redis/3-7-other-commands/3-7-3-expiring-keys/) the data after a certain amount of time. Depending on your
application this may be appropriate or not. If there are lots of reads with
infrequent writes then a longer auto-expire time may be appropriate (such as 4
or 12 hours for example). Lots of writes may deserve a shorter time or may
justify not using a cache at all.

1. backendapp <- postgres
2. backendapp -> cache


* Case 2 -
If the database query result has been previously stored in the cache, then the
data is retrieved (much faster than a traditional db query since the database
access and compares values on disk and the cache does so from RAM). Then the db
query does not occur. Caches can be easier to horizontally scale than relational
databases and this strategy can be used to alleviate the burden on a single
relational database for a monolith for example.

1. backendapp <- cache


### Performance

Cache, typically being in Ram and consisting of a simple key/value query is
usually an order of [magnitude faster](https://redis.io/topics/benchmarks) than
a database query. Your mileage may vary depending on your stack. At the risk of
offending the internet I'll share antidotal evidence. My observations (network
trip included) on a recent application had cache results being returned in
approx 5-20 miliseconds and Sql database queries typically between 100
miliseconds and 2000 miliseconds. However, it is worth mentioning that many of
these queries were quite complex so again you will have to benchmark your own
app to be certain.

## Usage 

```golang

// MovieName stores the name of the Monty Python movie
type MovieName struct {
	TableID    int    `db:"table_id"`
	DataString string `db:"data_string"`
}

// GetMovieName queries the name of the Monty Python voie
func GetMovieName(movieID int) (rtn MovieName) {

	var err error

	QueryKeyFormat := "simpletable:%d"

	queryKey := fmt.Sprintf(QueryKeyFormat, movieID)

	// If there is a cache miss
	if database.IsCacheConnected() {
		err = database.GetRedisCacheCodec().Get(queryKey, &rtn)
	}

	if err != nil || !database.IsCacheConnected() {

		query := `select
    		       table_id,
    		       data_string
    	       from
    		       simpletable
    	       where
    		       table_id = :table_id`

		// Get data from db
		nstmt, err := database.GetDB().PrepareNamed(query)

		if err != nil {
			log.Fatal(err)
		}

		args := map[string]interface{}{
			"table_id": movieID,
		}

		err = nstmt.Get(&rtn, args)

		if err != nil {
			log.Fatal(err)
		}

		if database.IsCacheConnected() {
			// Place data into cache for next time, expire in 1 hour
			err = database.GetRedisCacheCodec().Set(&redisCache.Item{
				Key:        queryKey,
				Object:     rtn,
				Expiration: time.Hour,
			})
			if err != nil {
				panic(err)
			}
		}
		log.Debug("DB hit GetMovieName")
	} else {
		log.Debug("Cache hit GetMovieName")
	}

	return
}

```

## Setup

### Build and run postgres db

1. This Builds the local postgres container with two local SQL files that create a table
and insert some sample data into the table

```
docker build -t postgres . -f deploy/Dockerfile
```

2. Run the database. DON'T deploy anything with this simple default password,
this is for local learning ONLY.

```
docker run --name some-postgres -e POSTGRES_PASSWORD=mysecretpassword  -p 5432:5432 -d  postgres
```

### Run Redis

Redis can be run with the following:

```
docker run --name some-redis -p 6379:6379 -d redis
```


### Build and run the code

```
go build && ./db-cache
```


### Curl endpoint

Open a new terminal tab and curl the endpoint

```
curl -X GET "http://localhost:8080/helloget/2" -H "accept: application/json"
```

You should see

```
{"Hello":{"TableID":2,"DataString":"Monty Pythons Life of Brian"}}
```

In one tab and the other you should see something similar

```
INFO[0001]/home/pl/go/src/github.com/peterlamar/go-examples/db-cache/api/api.go:47 github.com/peterlamar/go-examples/db-cache/api.Helloget() GetMovieName DB took 5.765818ms              
INFO[0001]/home/pl/go/src/github.com/peterlamar/go-examples/db-cache/api/api.go:54 github.com/peterlamar/go-examples/db-cache/api.Helloget() GetMovieName Cache took 823.317µs
```

The cache is performing faster than the sql DB in this instance. Curl it again
and you sould see the db performance improve as it also has an internal cache

```
INFO[0002]/home/pl/go/src/github.com/peterlamar/go-examples/db-cache/api/api.go:47 github.com/peterlamar/go-examples/db-cache/api.Helloget() GetMovieName DB took 1.055244ms              
INFO[0002]/home/pl/go/src/github.com/peterlamar/go-examples/db-cache/api/api.go:54 github.com/peterlamar/go-examples/db-cache/api.Helloget() GetMovieName Cache took 653.042µs  
```

### Optional

Postgres connection vars are optional and may be set as follows. Make sure
any modifications are matched in the postgres dockerfile and connection string.

```
export POSTGRES_HOST=localhost
export POSTGRES_DB=database-name
export POSTGRES_USER=database-user-name
export POSTGRES_PASSWORD=super-secret-password
export CONNECTION_TIMEOUT=duration before connection timeout when starting
 connection to db
export POSTGRES_PORT=database-port
```

Custom Redis ports can be setup with:

```
export REDIS_PORT=redis-port
```

#### Investigate postgres db

1.Launch psql to connect to postgres and look around. You will need
to enter the postgres password which would be "mysecretpassword" if you copy
and pasted above

```
docker run -it --rm --link some-postgres:postgres postgres psql -h postgres -U postgres
```

You should see:

```
postgres
Password for user postgres:
psql (10.6)
Type "help" for help.

postgres=#
```

2. In psql

```
select * from public.simpletable;
```

You should see:
```
table_id |                data_string                 
----------+--------------------------------------------
       1 | Monty Python and the Holy Grail
       2 | Monty Pythons Life of Brian
       3 | Monty Python Live at the Hollywood Bowl
       4 | Monty Pythons The Meaning of Life
       5 | And Now for Something Completely Different
(5 rows)
```

### References

[Postgres docker hub](https://hub.docker.com/_/postgres)
[db caching layer](https://www.reddit.com/r/golang/comments/6o8rzt/how_do_you_organize_db_and_caching_layer/)
