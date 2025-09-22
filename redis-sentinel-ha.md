# Method for Redis (Bare-Metal)

**Redis replication + Sentinel + HAProxy** guide for your 3 nodes.

> Topology
> 
> - node-1 (192.168.1.10) — **master**
> - node-2 (192.168.1.11) — **replica**
> - node-3 (192.168.1.12) — **replica**
> 
> **Ports**: open **6379** (Redis) and **26379** (Sentinel) on all nodes.
> 
> **Password (example)**: `A_STRONG_PASSWORD`
> 

---

# 1) Install Redis & Sentinel (all nodes)

```bash
sudo apt-get update
sudo apt-get install -y redis-server redis-sentinel
```

Enable services (we’ll restart after config):

```bash
sudo systemctl enable redis-server
sudo systemctl enable redis-sentinel
```

---

# 2) Configure Redis — Master (node-1)

Edit `/etc/redis/redis.conf`:

```
# Networking
bind 192.168.1.10
port 6379
protected-mode yes
supervised systemd

# Security
requirepass A_STRONG_PASSWORD
# master does NOT need masterauth

# Persistence (recommended)
appendonly yes
appendfsync everysec
# keep default RDB saves or adjust:
save 900 1
save 300 10
save 60 10000

# Optional hardening (uncomment to use)
# rename-command FLUSHALL ""
# rename-command FLUSHDB  ""
# rename-command CONFIG   ""
# rename-command KEYS     ""
```

Restart & check:

```bash
sudo systemctl restart redis-server
sudo systemctl status redis-server --no-pager
```

---

# 3) Configure Redis — Replicas (node-2 & node-3)

Edit `/etc/redis/redis.conf` on **each replica** with its own IP:

```
# (node-2)
bind 192.168.1.11
port 6379
protected-mode yes
supervised systemd

requirepass A_STRONG_PASSWORD
masterauth A_STRONG_PASSWORD

# Follow the master (node-1)
replicaof 192.168.1.10 6379

# Persistence
appendonly yes
appendfsync everysec
save 900 1
save 300 10
save 60 10000
```

```
# (node-3)
bind 192.168.1.12
port 6379
protected-mode yes
supervised systemd

requirepass A_STRONG_PASSWORD
masterauth A_STRONG_PASSWORD

replicaof 192.168.1.10 6379

appendonly yes
appendfsync everysec
save 900 1
save 300 10
save 60 10000
```

Restart & check both:

```bash
sudo systemctl restart redis-server
sudo systemctl status redis-server --no-pager
```

**Verify replication:**

```bash
# On a client or any node:
redis-cli -h 192.168.1.10 -a A_STRONG_PASSWORD INFO replication | head -n 20
redis-cli -h 192.168.1.11 -a A_STRONG_PASSWORD ROLE
redis-cli -h 192.168.1.12 -a A_STRONG_PASSWORD ROLE
# Expect: node-1 -> master, node-2/3 -> slave/replica (online)
```

---

# 4) Configure Sentinel (all nodes)

> Sentinel should run on all three nodes, each watching the same master.
> 

Edit `/etc/redis/sentinel.conf` **on each node**, adjusting `bind` to the node’s IP:

```
# Listen
port 26379
bind 192.168.1.10         # node-1 (use 192.168.1.11 on node-2, 192.168.1.12 on node-3)
protected-mode no          # Sentinel has no auth; rely on host firewall

# Monitor the master
sentinel monitor mymaster 192.168.1.10 6379 2

# Auth to talk to Redis master/replicas
sentinel auth-pass mymaster A_STRONG_PASSWORD

# Timings and sync behavior
sentinel down-after-milliseconds mymaster 3000
sentinel failover-timeout mymaster 8000
sentinel parallel-syncs mymaster 1

# Optional if multiple NICs / NAT:
# sentinel announce-ip <this-node-ip>
# sentinel announce-port 26379
```

Restart & check on **each node**:

```bash
sudo systemctl restart redis-sentinel
sudo systemctl status redis-sentinel --no-pager
```

**Verify Sentinel cluster view (from any node):**

```bash
redis-cli -h 192.168.1.10 -p 26379 INFO sentinel
# Check that:
#   - sentinel_masters:1
#   - master0 shows address=192.168.1.10:6379
#   - sentinels=3 and slaves=2
```

---

# 5) Test automatic failover

1. Stop master (simulate outage):

```bash
sudo systemctl stop redis-server   # on node-1
```

1. Watch Sentinel:

```bash
redis-cli -h 192.168.1.11 -p 26379 INFO sentinel | grep master0
redis-cli -h 192.168.1.12 -p 26379 INFO sentinel | grep master0
```

You should see a new master elected (either node-2 or node-3).

1. Verify roles:

```bash
redis-cli -h 192.168.1.11 -a A_STRONG_PASSWORD ROLE
redis-cli -h 192.168.1.12 -a A_STRONG_PASSWORD ROLE
# One should now be "master"
```

1. Bring node-1 back:

```bash
sudo systemctl start redis-server    # node-1 rejoins as a replica automatically
```

---

# 6) HAProxy in front of Redis master (Optional)

Install HAProxy on a separate box or one of the nodes:

```bash
sudo apt-get install -y haproxy
```

Example `/etc/haproxy/haproxy.cfg` that **routes only to the current master** (health-checked via AUTH + INFO role):

```
global
    log /dev/log local0
    log /dev/log local1 notice
    user haproxy
    group haproxy
    daemon
    nbthread 4
    maxconn 40000

defaults
    log global
    mode tcp
    option tcplog
    option dontlognull
    timeout connect 5s
    timeout client  50s
    timeout server  50s

frontend fe_redis
    bind *:6380
    default_backend be_redis_master

backend be_redis_master
    mode tcp
    balance first
    option tcp-check

    # Health check sequence: AUTH -> PING -> INFO role:master -> QUIT
    tcp-check send "AUTH A_STRONG_PASSWORD\r\n"
    tcp-check expect string +OK
    tcp-check send "PING\r\n"
    tcp-check expect string +PONG
    tcp-check send "INFO replication\r\n"
    tcp-check expect string role:master
    tcp-check send "QUIT\r\n"
    tcp-check expect string +OK

    server redis1 192.168.1.10:6379 maxconn 20000 check inter 2s fall 3 rise 1
    server redis2 192.168.1.11:6379 maxconn 20000 check inter 2s fall 3 rise 1
    server redis3 192.168.1.12:6379 maxconn 20000 check inter 2s fall 3 rise 1

```

Restart HAProxy:

```bash
sudo systemctl restart haproxy
sudo systemctl status haproxy --no-pager

```

**Clients connect via HAProxy:**

```bash
redis-cli -h <haproxy_ip> -p 6380 -a A_STRONG_PASSWORD PING

```

---

# 7) Useful verification commands

```bash
# Replication state (master or replica)
redis-cli -h 192.168.1.10 -a A_STRONG_PASSWORD INFO replication

# Current master via Sentinel
redis-cli -h 192.168.1.11 -p 26379 SENTINEL get-master-addr-by-name mymaster

# All Sentinel masters/slaves/sentinels
redis-cli -h 192.168.1.12 -p 26379 SENTINEL masters
redis-cli -h 192.168.1.12 -p 26379 SENTINEL slaves mymaster
redis-cli -h 192.168.1.12 -p 26379 SENTINEL sentinels mymaster

```
