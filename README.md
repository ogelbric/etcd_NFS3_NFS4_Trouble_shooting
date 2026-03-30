# etcd_NFS3_NFS4_Trouble_shooting

# Message 
```
apply request took too long

```
# Simple Disk speed test
#Linux
```
dd if=/dev/zero of=/tmp/test.img bs=4k count=256k oflag=direct
```

#Photon OS
```
dd if=/dev/zero of=/var/lib/etcd/test.img bs=4k count=10000 oflag=dsync
```

Interpreting the Metric
Ideal Value: The 99th percentile () of wal_fsync should be well under 10ms.
High Value Indication: If this metric is high, the disk is too slow, causing backend commit issues (etcd_disk_backend_commit_duration_seconds) and potential cluster instability.
Action: If wal_fsync latency increases significantly, it likely indicates a need for faster disks (I/O provisioning)

# etcd_disk_wal_fsync_duration_seconds
#Disk fsync latency (etcd_disk_wal_fsync_duration_seconds): Crucial for etcd stability. Should be at p99.
watch -n 1 'curl -s http://localhost:2381/metrics | grep etcd_disk_wal_fsync_duration_seconds'
#
# etcd_disk_backend_commit_duration_seconds
#Backend commit latency (etcd_disk_backend_commit_duration_seconds): Time taken to commit data to the backend.
watch -n 1 'curl -s http://localhost:2381/metrics | grep etcd_disk_backend_commit_duration_seconds'
#
# etcd_network_peer_round_trip_time_seconds
#Peer round-trip time (etcd_network_peer_round_trip_time_seconds): Measures network latency between etcd members.
watch -n 1 'curl -s http://localhost:2381/metrics | grep etcd_network_peer_round_trip_time_seconds'

# Testing etcd Disk Latency with fio (NFS) 
#Run this from the control plane node targeting your NFS mount to simulate etcd's WAL writing: 
fio --rw=write --ioengine=sync --fdatasync=1 --directory=/path/to/nfs/mount --size=22m --bs=2300 --name=etcd-test

# Checking NFS Specific Latency
nfsiostat 1
nfsstat -c

# This etcd_server_slow_apply_total is the smoking gun on storage latency. 
```
curl -s http://localhost:2381/metrics | \
  grep -E "etcd_server_leader_changes\
|etcd_server_heartbeat_send_failures\
|etcd_server_slow_apply_total\
|etcd_server_slow_read_indexes_total"
```
```
# HELP etcd_server_heartbeat_send_failures_total The total number of leader heartbeat send failures (likely overloaded from slow disk).
# TYPE etcd_server_heartbeat_send_failures_total counter
etcd_server_heartbeat_send_failures_total 6
# HELP etcd_server_leader_changes_seen_total The number of leader changes seen.
# TYPE etcd_server_leader_changes_seen_total counter
etcd_server_leader_changes_seen_total 2
# HELP etcd_server_slow_apply_total The total number of slow apply requests (likely overloaded from slow disk).
# TYPE etcd_server_slow_apply_total counter
etcd_server_slow_apply_total 954
# HELP etcd_server_slow_read_indexes_total The total number of pending read indexes not in sync with leader's or timed out read index requests.
# TYPE etcd_server_slow_read_indexes_total counter
etcd_server_slow_read_indexes_total 13
```

# etcd_disk_wal_fsync
```
curl -s http://localhost:2381/metrics | grep etcd_disk_wal_fsync_duration_seconds_bucket
```

# nConnect is set to 4. 

```
esxcli storage nfs param set -v <volume-label> -c <number_of_connections>
```
or
```
esxcfg-advcfg -s 8 /NFS41/MaxNConnectConns          
esxcfg-advcfg -s 8 /NFS/MaxConnectionsPerDatastore
```
