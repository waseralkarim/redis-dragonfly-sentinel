# HAProxy in front of Redis/Dragonfly master (Optional)

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
