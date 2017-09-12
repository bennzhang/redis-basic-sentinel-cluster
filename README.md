# redis-basic-sentinel-cluster
# Redis Simple Instance

## Install Redis on Centos
```
$ sudo yum install epel-release
$ sudo yum update
$ sudo yum install redis
```
For latest version of Redis,[Check Redis website](https://redis.io/download)

#### start/stop redis
```
$ sudo systemctl start redis
$ sudo systemctl stop redis
```

#### check connection
```
$ redis-cli ping
PONG

# remote server check 
$ redis-cli -h <REMOTE_SERVER_IP> -p 6379 ping
PONG


$ redis-cli
redis> set foo bar
OK
redis> get foo
"bar"
```

#### check redis version
```
$ redis-server --version
Redis server v=3.2.0 sha=00000000:0 malloc=jemalloc-4.0.3 bits=64 build=8d88dc6cfd069418
```


## Configure Redis
Edit `/etc/redis.conf` file, after configure, restart redis `sudo systemctl restart redis`.

#### Change bind address to allow external connections
comment out this line
```
# bind 127.0.0.1
```
Change to allow all external connections or limited ips here 
```
# allow all connections or limited ips here 
bind 0.0.0.0
```

#### Add password
uncomment this line and change  `foobared` to a strong password
```
# requirepass YOURSTRONGPASS
```
Test
```
$ redis-cli
127.0.0.1:6379> set foo bar
(error) NOAUTH Authentication required.
127.0.0.1:6379> auth YOURSTRONGPASS
OK
```

#### Persistence Options
Redis provides two options for disk persistence: One is Point-in-time snapshots of the dataset, made at specified intervals (RDB).
The other is Append-only logs (AOF) and it works by copying incoming write to disk as they happen. Those two options can work together or separately.Below is an example of settings. [Check more details on Redis website](http://redis.io/topics/persistence)

```
# Snapshot persistence options
save 60 1000
stop-writes-on-bgsave-error no
rdbcompression yes
dbfilename dump.rdb

# Append-Only file 
appendonly yes
appendfsync everysec
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

# where to store
dir ./
```
**RDB Option:**

- `save 60 1000`: Redis will automatically trigger a `BGSAVE`
operation if 1,000 writes have occurred within 60 seconds.  You can also write multiple `save` lines in the configuration file `save 900 1`, `save 300 10` and `save 60 10000`. If any one of rules match, a `BGSAVE` will be triggered.  Any Redis client can initiate a snapshot by calling the `BGSAVE` command. When `BIGSAVE` is triggered,  Redis forks a child process and the child process will write the snapshot to disk while the parent process continues to respond to commands.

- `stop-writes-on-bgsave-error no` : Redis will stop accpeting writes when there are problems with `bgsave`.

- `rdbcompression yes`: Needs a little more CPU but reduces the filesize.

- `dbfilename dump.rdb`: Choose the name of your snapshot file
- `rdbchecksum no`: Choose to creates a CRC64 checksum at the end of the snapshot file or not. 
- `dir /var/lib/redis/`: Choose the directory where to save the snapshot file.

**AOF Option:**

- `appendonly yes`: Default is `no`. To enable, set to `yes`

- `appendfsync everysec` 
   - everysec: fsync every second. 
   - no: don't fsync and let OS choose when to FLUSH the output. It has the best performance.
   - always: fsync for every write.
- `no-appendfsync-on-rewrite no`: 
   - no: (data safety, but produces additional I/O)
   - yes: (outage during aof-rewrite will cause data-loss, lower I/O)
- `auto-aof-rewrite-percentage 100`: When the size exceeds 100% the rewrite is triggered. Set it to “0” to disable the rewrite completely
- `auto-aof-rewrite-min-size 64mb`: Min Size of the File to trigger the rewrite (even if percentage is reached)


## Backup and Restore
You need to periodically backup/encrypt/transfer out the AOF logs or RDB snapshots. To Restore the backups, stop the Redis server and copy the file to the data-dir location (/var/lib/redis) and restart the Redis. The dataset is loaded from the file for RDB. For AOF, Redis will execute all commands from the AOF log.


# Redis Sentinel
`Redis Sentinel` provides high availability of Redis. It provides `monitor`, `notification`, `automatic failover` and etc. [Check more on Redis website](https://redis.io/topics/sentinel)

Below simply describes how to setup `Redis Sentinel`.

### Create configurations
Copy sample redis configuration files from `/etc/` folder to `sentinel`  

```
cp /etc/redis.conf sentinel/
cp /etc/redis-sentinel.conf sentinel/
```

Create this tree structure of configurations like below.  The `redis.conf` under the base folder `sentinel` is basic configuration file. And it is included in master configure: `node-6379/redis.conf` and two slave configurations: `node-6380/redis.conf` and `node-6381/redis.conf`. The difference between master and slave `redis.conf` are `port number` and the extra line `slaveof 127.0.0.1 6379` in the slave configuration files. 

Don't forget to change the include path to the **`/absolute/path/to`** of `sentinel` folder in all nodes `redis.conf` configuration files. 

```
$ tree sentinel
sentinel
├── node-6379
│   ├── redis.conf
│   ├── sentinel.conf
│   └── state
│       └── dump.rdb
├── node-6380
│   ├── redis.conf
│   ├── sentinel.conf
│   └── state
│       └── dump.rdb
├── node-6381
│   ├── redis.conf
│   ├── sentinel.conf
│   └── state
│       └── dump.rdb
├── redis.conf
├── redis-sentinel.conf
├── start-all.sh
└── stop-all.sh

```
`redis.conf` and `sentinel.conf` under `node-6379` looks like

```
$ cat node-6379/redis.conf
include /absolute/path/to/sentinel/redis.conf
port 6379
requirepass "yourpassword"
dir /absolute/path/to/sentinel/node-6379/state

$ cat node-6380/redis.conf
include /absolute/path/to/sentinel/redis.conf
port 6380
masterauth "yourpassword"
dir /absolute/path/to/sentinel/node-6380/state
slaveof 127.0.0.1 6379

$ cat node-6381/redis.conf
include /absolute/path/to/sentinel/redis.conf
port 6381
masterauth "yourpassword"
dir /absolute/path/to/sentinel/node-6381/state
slaveof 127.0.0.1 6379
```

For `sentinel.conf` configurations, they are all the same but `port number`

```
$ cat node-6379/sentinel.conf
port 26379
dir "/absolute/path/to/sentinel/node-6379/state"
sentinel monitor mymaster 127.0.0.1 6379 2
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 60000
sentinel parallel-syncs mymaster 1
sentinel auth-pass mymaster yourpassword
```

### Start redis and sentinel servers

Start all redis and sentinel server by running `start-all.sh` 
```
$ cat start-all.sh
redis-server node-6379/redis.conf &
redis-sentinel node-6379/sentinel.conf &

redis-server node-6380/redis.conf &
redis-sentinel node-6380/sentinel.conf &

redis-server node-6381/redis.conf &
redis-sentinel node-6381/sentinel.conf &
``` 

### Check processes and connections:

Check redis processes

```
$ ps -ef | grep redis
root      5214     1  0 16:05 pts/0    00:00:00 redis-server 127.0.0.1:6379
root      5215     1  0 16:05 pts/0    00:00:00 redis-sentinel *:26379 [sentinel]
root      5216     1  0 16:05 pts/0    00:00:00 redis-server 127.0.0.1:6380
root      5217     1  0 16:05 pts/0    00:00:00 redis-sentinel *:26380 [sentinel]
root      5218     1  0 16:05 pts/0    00:00:00 redis-server 127.0.0.1:6381
root      5219     1  0 16:05 pts/0    00:00:00 redis-sentinel *:26381 [sentinel]
```

Check redis connections info. [More commands on Redis Website](https://redis.io/topics/sentinel)

```
$ redis-cli -p 6379 -a yourpassword ping
PONG
$ redis-cli -p 6380 -a yourpassword ping
PONG
$ redis-cli -p 6381 -a yourpassword ping
PONG

$ redis-cli -p 26379 -a yourpassword sentinel master mymaster
  1) "name"
 2) "mymaster"
 3) "ip"
 4) "127.0.0.1"
 5) "port"
 6) "6379"
 7) "runid"
 8) "bee34e3dfa794613e074d1985dc867ce6f264575"
 9) "flags"
10) "master"
11) "link-pending-commands"
12) "0"
13) "link-refcount"
14) "1"
15) "last-ping-sent"
16) "0"
17) "last-ok-ping-reply"
18) "630"
19) "last-ping-reply"
20) "630"
21) "down-after-milliseconds"
22) "5000"
23) "info-refresh"
24) "8696"
25) "role-reported"
26) "master"
27) "role-reported-time"
28) "129204"
29) "config-epoch"
30) "0"
31) "num-slaves"
32) "2"
33) "num-other-sentinels"
34) "2"
35) "quorum"
36) "2"
37) "failover-timeout"
38) "60000"
39) "parallel-syncs"
40) "1"


$  redis-cli -p 26379 -a yourpassword sentinel sentinels mymaster
1)  1) "name"
    2) "cd65b64ad5032426b9f82ba94e790339b5c107ab"
    3) "ip"
    4) "127.0.0.1"
    5) "port"
    6) "26380"
    7) "runid"
    8) "cd65b64ad5032426b9f82ba94e790339b5c107ab"
    9) "flags"
   10) "sentinel"
   11) "link-pending-commands"
   12) "0"
   13) "link-refcount"
   14) "1"
   15) "last-ping-sent"
   16) "0"
   17) "last-ok-ping-reply"
   18) "67"
   19) "last-ping-reply"
   20) "67"
   21) "down-after-milliseconds"
   22) "5000"
   23) "last-hello-message"
   24) "151"
   25) "voted-leader"
   26) "?"
   27) "voted-leader-epoch"
   28) "0"
2)  1) "name"
    2) "2d4f4f76c4f76a3481a5918301b2da01b51e9a71"
    3) "ip"
    4) "127.0.0.1"
    5) "port"
    6) "26381"
    7) "runid"
    8) "2d4f4f76c4f76a3481a5918301b2da01b51e9a71"
    9) "flags"
   10) "sentinel"
   11) "link-pending-commands"
   12) "0"
   13) "link-refcount"
   14) "1"
   15) "last-ping-sent"
   16) "0"
   17) "last-ok-ping-reply"
   18) "67"
   19) "last-ping-reply"
   20) "67"
   21) "down-after-milliseconds"
   22) "5000"
   23) "last-hello-message"
   24) "342"
   25) "voted-leader"
   26) "?"
   27) "voted-leader-epoch"
   28) "0"


$ redis-cli -p 26379 -a yourpassword sentinel get-master-addr-by-name mymaster
1) "127.0.0.1"
2) "6379"
```

### Test failover
```
redis-cli -p 26379 -a yourpassword
127.0.0.1:26379> info
```

Below only capture `Sentinel` info, see the master is `6379`

```
# Sentinel
sentinel_masters:1
sentinel_tilt:0
sentinel_running_scripts:0
sentinel_scripts_queue_length:0
sentinel_simulate_failure_flags:0
master0:name=mymaster,status=sdown,address=127.0.0.1:6379,slaves=2,sentinels=3
```

Run this command to force failover.
```
127.0.0.1:26379> SENTINEL failover mymaster
```

See the progress in the logs

```
5215:X 12 Sep 16:13:49.237 * +slave-reconf-inprog slave 127.0.0.1:6381 127.0.0.1 6381 @ mymaster 127.0.0.1 6379
5217:X 12 Sep 16:13:58.311 * +convert-to-slave slave 127.0.0.1:6379 127.0.0.1 6379 @ mymaster 127.0.0.1 6380
5215:X 12 Sep 16:14:30.293 # +sdown master mymaster 127.0.0.1 6379
5215:X 12 Sep 16:14:48.201 # +failover-end-for-timeout master mymaster 127.0.0.1 6379
5215:X 12 Sep 16:14:48.201 # +failover-end master mymaster 127.0.0.1 6379
5215:X 12 Sep 16:14:48.201 * +slave-reconf-sent-be slave 127.0.0.1:6380 127.0.0.1 6380 @ mymaster 127.0.0.1 6379
5215:X 12 Sep 16:14:48.201 * +slave-reconf-sent-be slave 127.0.0.1:6381 127.0.0.1 6381 @ mymaster 127.0.0.1 6379
5215:X 12 Sep 16:14:48.201 # +switch-master mymaster 127.0.0.1 6379 127.0.0.1 6380
5215:X 12 Sep 16:14:48.201 * +slave slave 127.0.0.1:6381 127.0.0.1 6381 @ mymaster 127.0.0.1 6380
5215:X 12 Sep 16:14:48.201 * +slave slave 127.0.0.1:6379 127.0.0.1 6379 @ mymaster 127.0.0.1 6380
```

We can see master was switched to `6380`, show `info` again
```
127.0.0.1:26379> info
# Sentinel
sentinel_masters:1
sentinel_tilt:0
sentinel_running_scripts:0
sentinel_scripts_queue_length:0
sentinel_simulate_failure_flags:0
master0:name=mymaster,status=ok,address=127.0.0.1:6380,slaves=2,sentinels=3
```


### Stop all 
Start all redis and sentinel server by running `stop-all.sh`
```
$ cat stop-all.sh
redis-cli -p 26381 SHUTDOWN NOSAVE
redis-cli -p 26380 SHUTDOWN NOSAVE
redis-cli -p 26379 SHUTDOWN NOSAVE

redis-cli -p 6381 SHUTDOWN NOSAVE
redis-cli -p 6380 SHUTDOWN NOSAVE
redis-cli -p 6379 SHUTDOWN NOSAVE
```

# Redis Cluster 
