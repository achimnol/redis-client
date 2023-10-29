# redis-client
An experimental async-native Redis client to improve HA support for modern Python


## Goals

* Provide a proper blocking connection pool implementation
* Distinguish different sources of timeouts
* Explicitly support retries for blocking commands
* Clear exception hierarchy: Client / Server (like aiohttp)

## Non-goals

* Backward compatibility with existing redis-py / aioredis API
* Support Python 3.10 or older - We are going to fully utilize the task scoping support in Python 3.11 when necessary
* Synchronous APIs, although we may add a compat layer in the future

## Ideation

[Refer the wiki.](https://github.com/achimnol/redis-client/wiki/Overall-API-scheme)
