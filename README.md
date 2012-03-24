# labs-redis-clojure

This is a Clojure client lib for [Redis](https://github.com/antirez/redis)

*Here be dragons!* .. or rather, Java. The client is a small core of java that implements the Redis protocol
with a clojure API for Redis commands around it.

## Status
*WORK IN PROGRESS!* I expect to have a stable release in a couple of weeks, but things seems to work ok.

## Features
- Pretty fast. The core Java part is in the same ballpark as redis-bench
- Commands are generated from [redis-doc](https://github.com/antirez/redis-doc) (commands.json). This means it's easy to
keep the client up-to-date with Redis development. Also, useful documention for command fns!
- All commands are pipelined by default and returns futures.
- Futures return can be deref:ed with `(deref xx)` and `@` (returns Reply)
- Replies can alse be deref:ed to their underlying values `@@(ping db) => "PONG"`
- Return values are processed as little as possible, eg. `@@(get db "xxx")` returns byte[].
Includes some helper fns for converting to `String` `(->str @(get r "xxx"))` and `String[]` (->strs)
- Sane pub/sub support, including correct behavior for UNSUBSCRIBE returning connection to normal state.
- Support for MULTI/EXEC and return values (see example below).
- labs-redis does not use global `*bindings*` for the connection ref (as in clj-redis and redis-clojure).
In my target code for this library talks to alot of different Redis instances and `(with-connection (client) (set key val))` adds alot of uneccesary boilerplate for us.
- Idiomatic support for EVAL. `defeval` handles caching of lua source per connection and EVALSHA use etc. `(defeval my-echo [] [x] "return redis.call('ECHO',ARGV[1])")`
- Small connection pool impl (`pool` and `with-pool` macro.) Work in progress..

## Basic Usage

```clojure
(require '[labs.redis.core :as redis])

(def db (redis/client))

(redis/ping db)
=> <ReplyFuture <StatusReply "PONG">>

@(redis/ping db)
=> <StatusReply "PONG">

(redis/value @(redis/ping db))
=> "PONG"

@@(redis/ping db)
=> "PONG"

@(redis/set db "foo" "bar")
=> <StatusReply "OK">

@(redis/get db "foo")
=> <BulkReply@xxxx: #<byte[] [B@xxxx]>>

@@(redis/get db "foo")
=> #<byte[] [B@xxxx]>

(String. @@(redis/get db "foo"))
=> "bar"

(redis/->str @(redis/get db "foo"))
=> "bar"
```

Arguments to commands are converted and flattened in a do-what-I-mean style, so we can write code like

```clojure
;; ZUNIONSTORE destination numkeys key [key ...] [WEIGHTS weight [weight ...]] [AGGREGATE SUM|MIN|MAX]
(zunionstore r "out" 2 ["set1" "set2"] {:weights [10 20] :aggregate :sum})
```

## PUB/SUB
```clojure
  (subscribe-with r
   ["msgs" "msgs2" "msgs3"]
   (fn [db channel message]
     (let [message (->str message)]
       (println "R " channel message)
       (not (= "quit" message)))))
```

## MULTI/EXEC
```clojure
  ;; multi/exec with sane handling of return values
  (try
    (multi r)
    (let [_ (mset r "k1" "v1" "k2" "v2")
          a (get r "k1")
	  _ (set r "k2" "xx")
          b (get r "k2")]
      (exec! r)
      (println (->str @a) (->str @b))) ;; v1 xx
    (catch Throwable t
      (try @(discard r)
           (finally (throw t)))))
```

There is also a macro `(atomically db  &body)` that does multi/exec/discard and return the MultiBulkReply from EXEC.
See source for details.

## What's missing

Tests..

## Installation / Leiningen

_Not deployed on clojars yet. Install locally or use leiningen checkouts_

Add `[labs-redis "0.1.0"]` in your `project.clj` then run `lein deps`.

## Author

Andreas Bielk :: andreas@preemptive.se :: @wallrat


## License

Copyright © 2012 Andreas Bielk

Distributed under the MIT/X11 license; see the file LICENSE.
