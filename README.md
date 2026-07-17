# pyredis

A from-scratch, in-memory key-value store — the Redis internals, without the
40 years of C and the marketing. Built to actually understand what a database
does under the hood, not just to consume one.

Speaks real **RESP** (Redis Serialization Protocol), the same wire format
Redis itself uses — so you can even point the real `redis-cli` at it.

## What's inside

| Concept | File | Notes |
|---|---|---|
| Wire protocol | `pyredis/protocol.py` | RESP encode/decode, shared by server, client, AOF, and replication |
| Memory + LRU eviction | `pyredis/store.py` | `OrderedDict`-backed store; every read/write moves a key to "most recently used"; oldest keys get evicted once you exceed `--maxkeys` |
| Persistence | `pyredis/aof.py` | Append-only log of every write command; replayed on startup; `BGREWRITEAOF` compacts it |
| Pub/Sub | `pyredis/pubsub.py` | Channel → subscriber-socket fan-out |
| TCP server + replication | `pyredis/server.py` | asyncio TCP server, command dispatch, leader→replica streaming |
| CLI | `pyredis/cli.py` | `redis-cli`-style REPL and one-shot command mode |

## Quick start

```bash
# Terminal 1: start the server
python3 -m pyredis.server --port 6380 --maxkeys 10000 --aof data.aof

# Terminal 2: talk to it
python3 -m pyredis.cli --port 6380
127.0.0.1:6380> SET name alice
OK
127.0.0.1:6380> GET name
"alice"
127.0.0.1:6380> SET session abc123 EX 60
OK
127.0.0.1:6380> TTL session
(integer) 60
```

One-shot mode (no REPL) also works, same as `redis-cli GET foo`:

```bash
python3 -m pyredis.cli --port 6380 GET name
```

**Bonus:** since this speaks real RESP, the actual `redis-cli` works against it too:
```bash
redis-cli -p 6380 SET foo bar
```

## Commands supported

```
PING [msg]              GET key                  SET key value [EX sec|PX ms]
DEL key [key...]        EXISTS key [key...]      EXPIRE key seconds
TTL key                 PERSIST key              KEYS pattern
DBSIZE                  FLUSHALL                  BGREWRITEAOF
INFO                    ECHO msg                  QUIT
SUBSCRIBE ch [ch...]    UNSUBSCRIBE [ch...]      PUBLISH ch message
REPLICAOF host port     REPLICAOF NO ONE
```

## The interesting parts

### LRU eviction (`store.py`)
Every `SET` or `GET` moves the key to the back of an `OrderedDict`. When the
store holds more than `--maxkeys` entries, we pop from the front — the least
recently touched key — until we're back under the limit. That's the entire
LRU algorithm; the trick is just choosing the right data structure so both
"move to most-recent" and "evict oldest" are O(1).

### Crash-safe persistence (`aof.py`)
Every write command is serialized in the exact RESP format a client would
send it, appended to a log file, and `fsync`'d immediately. On startup, the
log is replayed front-to-back to rebuild the dataset. I tested this with a
hard `kill -9` (no graceful shutdown) mid-session — the data was still there
after restart, TTLs included.

Because the log only grows, `BGREWRITEAOF` rewrites it down to the minimal
set of `SET` commands that reproduce the *current* live dataset (dropping
old, overwritten, or deleted keys) — same idea as Redis's AOF rewrite.

### Replication (`server.py`)
`REPLICAOF host port` on the replica opens a connection to the master and
sends `PSYNC`. The master responds with:
1. A **full sync** — every live key, re-encoded as `SET` commands.
2. Then it keeps that connection open and streams every future write
   command to it live, as they happen.

The replica applies each incoming command to its own store and its own AOF,
so it survives its own restarts too. This is single-leader, asynchronous,
non-failover replication — enough to see the actual mechanism (log
shipping) that real distributed databases build on.

### Why RESP instead of a made-up protocol
Implementing the real wire protocol (simple strings `+OK`, errors `-ERR`,
integers `:123`, bulk strings `$3\r\nfoo`, arrays `*2\r\n...`) means the
project produces something that's protocol-compatible with the real thing,
not just superficially similar.

## Try the eviction yourself

```bash
python3 -m pyredis.server --port 6380 --maxkeys 3 --aof /tmp/demo.aof
```
```bash
python3 -m pyredis.cli --port 6380 SET a 1
python3 -m pyredis.cli --port 6380 SET b 2
python3 -m pyredis.cli --port 6380 SET c 3
python3 -m pyredis.cli --port 6380 SET d 4   # a is now evicted, over the limit
python3 -m pyredis.cli --port 6380 KEYS "*"  # b, c, d
```

## Try replication yourself

```bash
# master
python3 -m pyredis.server --port 6400 --aof master.aof
# replica
python3 -m pyredis.server --port 6401 --aof replica.aof
python3 -m pyredis.cli --port 6401 REPLICAOF 127.0.0.1 6400
# now write on the master...
python3 -m pyredis.cli --port 6400 SET k v
# ...and read it back from the replica
python3 -m pyredis.cli --port 6401 GET k
```

## Known limitations (on purpose — this is a learning project, not prod)
- Single-threaded asyncio event loop — fine for learning, not for a
  production throughput benchmark.
- No AUTH / TLS.
- No failover / leader election if the master dies.
- LRU eviction is by key *count*, not actual byte-size of values (real Redis
  supports `maxmemory` in bytes — an easy follow-up extension).
- Only string values (no lists/hashes/sets — Redis's other data types would
  be a natural next milestone).
