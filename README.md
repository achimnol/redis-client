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

## Ideation (draft)

```python
import redis

client = await redis.Client(
    connection_pool_opts={  # using TypedDict
        "max_wait": ...,
        ...,
    },
    socket_opts={  # using TypedDict
        "connect_timeout": ...,
        "keepalive_interval": ...,
        ...,
    },
    ssl_options={...},
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
    async for attempt in client.retry(redis.RetryPolicy(...)):  # locally overridable
        response = await client.execute_blocking("xread", ...)
```

```python
import redis

client = await redis.SentinelClient(
    connection_pool_opts={...},
    socket_opts={...},
    retry_policy=redis.RetryPolicy(...),
)

client = await redis.ClusterClient(
    connection_pool_opts={...},
    socket_opts={...},
    retry_policy=redis.RetryPolicy(...),
)
```

> **Note**
> At the first stage, we will simply serialize and deserialize the request/response using the raw RESP3 spec instead of providing full functional wrappers of all Redis commands.

### Difficulties

* How do we distinguish user-specified intended timeouts in blocking commands and server-side timeouts (e.g., due to overloads)?
* Depending on the exceptions, we may need to perform additional actions or skip backoffs when retrying:
  - `ServerResponseTimeout` (due to blocking command): skip backoff
  - Recycle the connection object in the connection pool (close the current one, add a new one, and grab the created one)
    - If possible, we should reuse the connection object and the underlying socket to reduce the kernel context switching overhead.
