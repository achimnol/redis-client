# redis-client
An experimental async-native Redis client to improve HA support for modern Python


## Goals

* Provide a proper blocking connection pool implementation
* Distinguish different sources of timeouts
* Explicitly support retries for blocking commands

## Non-goals

* Backward compatibility with existing redis-py / aioredis API
* Support Python 3.10 or older - We are going to fully utilize the task scoping support in Python 3.11 when necessary
* Synchronous APIs, although we may add a compat layer in the future

## Ideation (draft)

```python
import redis

client = await redis.Client(
    connection_pool_opts={
        "max_wait": ...,
        ...,
    },
    socket_opts={  # using TypedDict
        "connect_timeout": ...,
        "keepalive_interval": ...,
        ...,
    },
    retry_policy=redis.RetryPolicy(  # (TODO) Backoff policys may differ by exception types.
        backoff=...,
        exceptions=[
            redis.ClientConnectionError,  # incl. ConnectionPoolTimeout, SocketTimeout
            redis.ServerBusyLoadingError,
            redis.ServerNoReplicaError,
            redis.ServerResposneTimeout,  # (TODO) In this case, we shouldn't apply backoff.
        ],
    ),
)

async with client:
    pipe = client.pipeline()
    pipe.execute("set", "myvar", "asdf")
    pipe.execute("push", "mylist", "zxcv")
    async for attempt in client.retry():
        await client.execute(pipe)

async with client:
    async for attempt in client.retry():
        response = await client.execute_blocking("xread", ...)
```

> **Note**
> At the first stage, we will simple serialize and deserialize the request/response using the raw RESP3 spec.
