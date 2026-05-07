# Ceph Commands Cheatsheet

Cheat sheet of common Ceph CLI commands for day‑to‑day admin, troubleshooting, and ops reference

---

## Table of Contents

1. [Deployment — cephadm](#1-deployment--cephadm)
2. [Bootstrap & Cluster Initialization](#2-bootstrap--cluster-initialization)
3. [Host Management](#3-host-management)
4. [OSD Management](#4-osd-management)
5. [Monitor (MON) Management](#5-monitor-mon-management)
6. [Manager (MGR) Management](#6-manager-mgr-management)
7. [Cluster Health & Status](#7-cluster-health--status)
8. [CRUSH Map & Topology](#8-crush-map--topology)
9. [Pool Management](#9-pool-management)
10. [Erasure Coding](#10-erasure-coding)
11. [RADOS & Object Storage](#11-rados--object-storage)
12. [RBD — Block Storage](#12-rbd--block-storage)
13. [CephFS — Filesystem](#13-cephfs--filesystem)
14. [RGW — Object Gateway (S3/Swift)](#14-rgw--object-gateway-s3swift)
15. [NFS Gateway](#15-nfs-gateway)
16. [iSCSI Gateway](#16-iscsi-gateway)
17. [Auth & Keyring Management](#17-auth--keyring-management)
18. [MDS — Metadata Server](#18-mds--metadata-server)
19. [PG (Placement Group) Management](#19-pg-placement-group-management)
20. [Monitoring & Alerting](#20-monitoring--alerting)
21. [Performance & Benchmarking](#21-performance--benchmarking)
22. [Scrubbing & Repair](#22-scrubbing--repair)
23. [Snapshots & Clones](#23-snapshots--clones)
24. [Quota Management](#24-quota-management)
25. [Configuration Management](#25-configuration-management)
26. [Orchestrator & Service Specs](#26-orchestrator--service-specs)
27. [Upgrade & Maintenance](#27-upgrade--maintenance)
28. [Disaster Recovery & Troubleshooting](#28-disaster-recovery--troubleshooting)
29. [Client Setup & Integration](#29-client-setup--integration)
30. [Tell / admin socket](#tell--admin-socket-quick-reference)
31. [Telemetry](#telemetry)

---

## 1. Deployment — cephadm

```bash
# Install cephadm (Debian/Ubuntu)
curl --silent --remote-name --location https://github.com/ceph/ceph/raw/quincy/src/cephadm/cephadm
chmod +x cephadm
sudo ./cephadm install

# Install cephadm (RHEL/CentOS/Rocky)
dnf install -y cephadm

# Add Ceph repo for a specific release
cephadm add-repo --release reef
cephadm add-repo --release quincy

# Install Ceph packages (after bootstrapping)
cephadm install ceph-common

# Pull the Ceph container image
cephadm pull

# Inspect running containers on a host
cephadm ls

# Enter the Ceph shell (containerized)
cephadm shell

# Run a specific Ceph command inside the container
cephadm shell -- ceph -s

# Check cephadm logs
journalctl -u ceph.target
journalctl -u "ceph-*"

# Adopt an existing non-containerized cluster
cephadm adopt --style legacy --name <daemon-name>

# Prepare a host for cephadm (install dependencies)
cephadm prepare-host

# Show cephadm and bundled Ceph versions
cephadm version

# Log in to a private container registry (air-gapped / mirror)
cephadm registry-login -i <path/to/docker.json>

# Run ceph-volume subcommands on a host (OSD create, zap, list)
cephadm ceph-volume -- <args>     # e.g. -- lvm list

# Remove a failed/partial cluster from this host (DESTRUCTIVE)
cephadm rm-cluster --fsid <fsid> --force
```

---

## 2. Bootstrap & Cluster Initialization

```bash
# Bootstrap a new cluster (minimal)
cephadm bootstrap --mon-ip <MON_IP>

# Bootstrap with a custom cluster network
cephadm bootstrap \
  --mon-ip <MON_IP> \
  --cluster-network <CIDR> \
  --public-network <CIDR>

# Bootstrap with a specific container image
cephadm bootstrap \
  --mon-ip <MON_IP> \
  --image quay.io/ceph/ceph:v18

# Bootstrap and skip dashboard (headless)
cephadm bootstrap --mon-ip <MON_IP> --skip-dashboard

# Bootstrap with initial dashboard credentials
cephadm bootstrap \
  --mon-ip <MON_IP> \
  --initial-dashboard-user admin \
  --initial-dashboard-password <PASSWORD>

# Bootstrap with an existing SSH key
cephadm bootstrap \
  --mon-ip <MON_IP> \
  --ssh-private-key /root/.ssh/id_rsa \
  --ssh-public-key /root/.ssh/id_rsa.pub

# Bootstrap with a custom FSID
cephadm bootstrap --mon-ip <MON_IP> --fsid <FSID>

# Bootstrap and apply a service spec (OSD/mon/RGW, etc.) immediately
cephadm bootstrap --mon-ip <MON_IP> --apply-spec initial-cluster.yaml

# Extra bootstrap options (common)
cephadm bootstrap --mon-ip <MON_IP> --ssh-config /root/.ssh/config
cephadm bootstrap --mon-ip <MON_IP> --skip-dashboard --skip-monitoring-stack
cephadm bootstrap --mon-ip <MON_IP> --allow-overwrite

# Generate a new FSID
python3 -c "import uuid; print(str(uuid.uuid4()))"

# Configure the Ceph CLI after bootstrapping
export CEPH_CONF=/etc/ceph/ceph.conf
export CEPH_KEYRING=/etc/ceph/ceph.client.admin.keyring
```

---

## 3. Host Management

```bash
# Add a host to the cluster
ceph orch host add <hostname> <IP>

# Add a host with labels
ceph orch host add <hostname> <IP> --labels _admin,mon,osd

# List all hosts
ceph orch host ls

# List hosts with a specific label
ceph orch host ls --label osd

# Add a label to an existing host
ceph orch host label add <hostname> <label>

# Remove a label from a host
ceph orch host label rm <hostname> <label>

# Set the maintenance mode for a host
ceph orch host maintenance enter <hostname>
ceph orch host maintenance exit <hostname>

# Drain a host (remove all daemons)
ceph orch host drain <hostname>

# Remove a host from the cluster
ceph orch host rm <hostname>

# Remove a host (force, even if daemons remain)
ceph orch host rm <hostname> --force

# Copy the SSH key to a host
ssh-copy-id -f -i /etc/ceph/ceph.pub root@<hostname>

# Check host connectivity
ceph cephadm check-host <hostname>

# Get facts about a host
ceph orch host ls --format json-pretty

# Rescan host devices (for new disks)
ceph orch device zap <hostname> <device_path> --force
ceph orch device ls <hostname>

# Change the IP cephadm should use for a host (after network change)
ceph orch host set-addr <hostname> <new-ip>
```

---

## 4. OSD Management

```bash
# List available devices on all hosts
ceph orch device ls

# List devices on a specific host
ceph orch device ls <hostname>

# List devices with refresh
ceph orch device ls --refresh

# Deploy OSDs on all available devices (auto)
ceph orch apply osd --all-available-devices

# Deploy OSDs from a YAML service spec
ceph orch apply -i osd-spec.yaml

# Create an OSD on a specific device
ceph orch daemon add osd <hostname>:<device_path>

# Zap (wipe) a device before OSD creation
ceph orch device zap <hostname> <device_path> --force

# List all OSDs
ceph osd ls

# Show OSD status
ceph osd stat

# Show detailed OSD info
ceph osd dump

# Show OSD tree
ceph osd tree

# Show OSD tree filtered to a specific host
ceph osd tree | grep -A5 <hostname>

# Mark an OSD down
ceph osd down <osd-id>

# Mark an OSD out (exclude from active sets — data migrates *off* this OSD)
ceph osd out <osd-id>

# Mark an OSD in
ceph osd in <osd-id>

# Remove an OSD gracefully (full workflow)
ceph osd out <osd-id>
ceph osd safe-to-destroy <osd-id>
ceph orch osd rm <osd-id>

# Destroy an OSD daemon
ceph orch daemon rm osd.<osd-id> --force

# Purge an OSD (remove from CRUSH + auth)
ceph osd purge <osd-id> --yes-i-really-mean-it

# Show OSD performance stats
ceph osd perf

# Show OSD utilization
ceph osd df
ceph osd df tree
ceph osd utilization              # summary vs cluster average

# OSDs blocking peering / recovery
ceph osd blocked-by

# Show OSD metadata
ceph osd metadata <osd-id>

# Reweight an OSD
ceph osd reweight <osd-id> <weight>

# Prefer / avoid this OSD as primary (0.0–1.0; does not change CRUSH weight)
ceph osd primary-affinity <osd-id> <0.0-1.0>

# Reweight OSDs by utilization
ceph osd reweight-by-utilization

# Set OSD flags
ceph osd set norebalance
ceph osd set nobackfill
ceph osd set norecover
ceph osd set noout
ceph osd set noscrub
ceph osd set nodeep-scrub
ceph osd set pause              # pause all IO

# Unset OSD flags
ceph osd unset norebalance
ceph osd unset nobackfill
ceph osd unset norecover
ceph osd unset noout
ceph osd unset noscrub
ceph osd unset nodeep-scrub
ceph osd unset pause

# Show OSD flags
ceph osd dump | grep flags

# Find which OSD hosts a specific object
ceph osd map <pool> <object-name>

# OSD class (device type) info
ceph osd crush class ls
ceph osd crush class create <class>      # e.g. ssd, hdd, nvme
ceph osd crush class rm <class>
ceph osd crush set-device-class <class> <osd-id> [<osd-id> ...]
ceph osd crush rm-device-class <osd-id> [<osd-id> ...]

# OSD release / compatibility (run before/after upgrades)
ceph osd versions
ceph osd require-osd-release <release>   # e.g. reef, squid
ceph osd set-require-min-compat-client <release>

# Safe to stop OSDs without breaking immediate availability (rollouts)
ceph osd ok-to-stop --ids=<osd-id> [<osd-id> ...] [--max <n>]
```

---

## 5. Monitor (MON) Management

```bash
# List monitors
ceph mon stat
ceph mon dump

# Add a monitor daemon
ceph orch daemon add mon <hostname>

# Apply monitor spec (set count / placement)
ceph orch apply mon 3                           # shorthand count (when supported)
ceph orch apply --service_type mon --placement="count:3"

# Apply monitors to specific hosts
ceph orch apply mon --placement="host1,host2,host3"

# Remove a monitor
ceph orch daemon rm mon.<hostname>

# Get monitor quorum status
ceph quorum_status
ceph quorum_status --format json-pretty

# Show monitor map
ceph mon getmap -o /tmp/monmap
monmaptool --print /tmp/monmap

# Compact the monitor store
ceph daemon mon.<hostname> compact

# Get monitor config at runtime
ceph daemon mon.<hostname> config show

# Inject a config key into the mon store (emergency)
ceph config-key set <key> <value>
ceph config-key get <key>
ceph config-key ls
```

---

## 6. Manager (MGR) Management

```bash
# List manager daemons
ceph mgr stat
ceph mgr dump

# Apply manager count
ceph orch apply mgr 2

# Add a manager daemon
ceph orch daemon add mgr <hostname>

# Show active manager
ceph mgr dump | grep active

# Failover active manager
ceph mgr fail <mgr-name>

# List available MGR modules
ceph mgr module ls

# Enable a module
ceph mgr module enable <module>         # e.g. dashboard, prometheus, pg_autoscaler

# Disable a module
ceph mgr module disable <module>

# Common modules
ceph mgr module enable dashboard
ceph mgr module enable prometheus
ceph mgr module enable pg_autoscaler
ceph mgr module enable balancer
ceph mgr module enable telemetry
ceph mgr module enable crash

# Configure balancer
ceph balancer mode upmap                  # preferred when OSDs support upmap
ceph balancer mode crush-compat           # legacy clusters / no upmap
ceph balancer mode none
ceph balancer on
ceph balancer off
ceph balancer status
ceph balancer eval
ceph balancer optimize <plan-name>
ceph balancer execute <plan-name>
```

---

## 7. Cluster Health & Status

```bash
# Quick cluster status
ceph -s
ceph status

# Cluster health summary
ceph health
ceph health detail

# Watch cluster status live
watch ceph -s

# Show all active warnings/errors
ceph health detail | grep -E "WARN|ERR"

# Show cluster version
ceph version
ceph versions

# Cluster FSID (matches fsid in ceph.conf / dashboards)
ceph fsid

# Show cluster config
ceph config dump

# Show cluster time sync status
ceph time-sync-status

# Crash reports
ceph crash ls
ceph crash ls-new
ceph crash info <crash-id>
ceph crash archive <crash-id>
ceph crash archive-all
ceph crash prune --keep <days>

# Cluster usage
ceph df
ceph df detail

# Show cluster features
ceph features

# Silence / unsilence health checks (maintenance windows)
ceph health mute <code> <ttl>           # e.g. OSD_DOWN 1h
ceph health mute <code> <ttl> --sticky  # until explicitly unmuted
ceph health unmute <code>

# Show progress of background operations
ceph progress
ceph progress json

# Show service map
ceph service map

# Show number of objects per pool
ceph df detail | grep -E "POOL|NAME"
```

---

## 8. CRUSH Map & Topology

```bash
# Get the compiled CRUSH map
ceph osd getcrushmap -o /tmp/crushmap.bin

# Decompile the CRUSH map to text
crushtool -d /tmp/crushmap.bin -o /tmp/crushmap.txt

# Edit and recompile
vi /tmp/crushmap.txt
crushtool -c /tmp/crushmap.txt -o /tmp/crushmap-new.bin

# Inject the new CRUSH map
ceph osd setcrushmap -i /tmp/crushmap-new.bin

# Test a CRUSH map (simulate placement)
crushtool --test -i /tmp/crushmap.bin --show-statistics \
  --rule <rule-id> --num-rep <count> --x <pg-id>

# List CRUSH rules
ceph osd crush rule ls
ceph osd crush rule dump

# Create CRUSH rules
ceph osd crush rule create-simple <rule-name> <root> <type>
ceph osd crush rule create-replicated <rule-name> <root> <failure-domain> <device-class>
ceph osd crush rule create-erasure <rule-name> [<profile>]

# Remove a CRUSH rule
ceph osd crush rule rm <rule-name>

# Add/move items in CRUSH
ceph osd crush add-bucket <name> <type>
ceph osd crush move <name> <type>=<parent>
ceph osd crush link <name> <type>=<parent>
ceph osd crush unlink <name> <ancestor>
ceph osd crush swap-bucket <src> <dst>
ceph osd crush set <osd-id> <weight> <type>=<bucket> [<type>=<bucket> ...]
ceph osd crush remove <name>

# Show CRUSH tree
ceph osd crush tree
ceph osd crush tree --show-shadow       # show device-class shadow trees

# Dump CRUSH map as JSON
ceph osd crush dump

# Show CRUSH weight
ceph osd crush show-tunables
ceph osd crush tunables optimal
ceph osd crush tunables default
```

---

## 9. Pool Management

```bash
# List pools
ceph osd pool ls
ceph osd pool ls detail

# Create a replicated pool
ceph osd pool create <pool-name> <pg_num>
ceph osd pool create <pool-name> <pg_num> <pgp_num> replicated

# Create an erasure-coded pool
ceph osd pool create <pool-name> <pg_num> <pgp_num> erasure [<ec-profile>]

# Set pool type to application
ceph osd pool application enable <pool-name> rbd
ceph osd pool application enable <pool-name> cephfs
ceph osd pool application enable <pool-name> rgw

# Remove application tag from a pool (break glass — confirm clients first)
ceph osd pool application disable <pool-name> <rbd|cephfs|rgw> \
  --yes-i-really-mean-it

# List pool applications
ceph osd pool application get <pool-name>

# Delete a pool (requires confirmation flags)
ceph osd pool delete <pool-name> <pool-name> --yes-i-really-really-mean-it

# Rename a pool
ceph osd pool rename <old-name> <new-name>

# Get pool stats
ceph osd pool stats
ceph osd pool stats <pool-name>

# Set pool parameters
ceph osd pool set <pool-name> size <n>              # replication factor
ceph osd pool set <pool-name> min_size <n>          # min replicas for IO
ceph osd pool set <pool-name> pg_num <n>            # number of PGs
ceph osd pool set <pool-name> pgp_num <n>
ceph osd pool set <pool-name> crush_rule <rule>
ceph osd pool set <pool-name> nodelete true
ceph osd pool set <pool-name> nopgchange true
ceph osd pool set <pool-name> nosizechange true
ceph osd pool set <pool-name> write_fadvise_dontneed true
ceph osd pool set <pool-name> bulk true             # pg_autoscaler hint

# Get pool parameter
ceph osd pool get <pool-name> <param>
ceph osd pool get <pool-name> all

# Set pool quota
ceph osd pool set-quota <pool-name> max_bytes <bytes>
ceph osd pool set-quota <pool-name> max_objects <n>

# Get pool quota
ceph osd pool get-quota <pool-name>

# Snapshot a pool (unmanaged)
ceph osd pool mksnap <pool-name> <snap-name>
ceph osd pool rmsnap <pool-name> <snap-name>
ceph osd pool ls detail | grep snap

# PG autoscaler
ceph osd pool set <pool-name> pg_autoscale_mode on
ceph osd pool set <pool-name> pg_autoscale_mode off
ceph osd pool set <pool-name> pg_autoscale_mode warn
ceph osd pool autoscale-status
```

---

## 10. Erasure Coding

```bash
# List erasure code profiles
ceph osd erasure-code-profile ls

# Get default profile
ceph osd erasure-code-profile get default

# Create a new EC profile
ceph osd erasure-code-profile set <profile-name> \
  k=<data-chunks> m=<coding-chunks> \
  crush-failure-domain=host \
  plugin=jerasure \
  technique=reed_sol_van

# Create an EC profile with LRC (Locally Repairable Codes)
ceph osd erasure-code-profile set <profile-name> \
  plugin=lrc \
  k=4 m=2 l=3 \
  crush-failure-domain=host

# Create an EC profile for SHEC
ceph osd erasure-code-profile set <profile-name> \
  plugin=shec \
  k=4 m=3 c=2

# Create an EC profile for ISA-L (Intel)
ceph osd erasure-code-profile set <profile-name> \
  plugin=isa \
  k=4 m=2 \
  crush-failure-domain=host

# Delete an EC profile
ceph osd erasure-code-profile rm <profile-name>

# Create EC pool with a profile
ceph osd pool create <pool-name> <pg_num> <pgp_num> erasure <profile-name>

# Enable overwrites on EC pool (required for RBD/CephFS)
ceph osd pool set <pool-name> allow_ec_overwrites true
```

---

## 11. RADOS & Object Storage

```bash
# List objects in a pool
rados -p <pool-name> ls
rados -p <pool-name> ls --all           # include snapshots

# Put (upload) an object
rados -p <pool-name> put <object-name> <file>

# Get (download) an object
rados -p <pool-name> get <object-name> <output-file>

# Delete an object
rados -p <pool-name> rm <object-name>

# Copy an object
rados -p <pool-name> cp <src-object> <dst-object>

# Check object info
rados -p <pool-name> stat <object-name>

# List object extended attributes (xattrs)
rados -p <pool-name> listxattr <object-name>

# Get/set/remove xattr
rados -p <pool-name> getxattr <object-name> <key>
rados -p <pool-name> setxattr <object-name> <key> <value>
rados -p <pool-name> rmxattr <object-name> <key>

# List/get/set/remove omap (key-value metadata)
rados -p <pool-name> listomapkeys <object-name>
rados -p <pool-name> getomapval <object-name> <key>
rados -p <pool-name> setomapval <object-name> <key> <value>
rados -p <pool-name> rmomapkey <object-name> <key>
rados -p <pool-name> clearomap <object-name>

# Lock/unlock an object
rados -p <pool-name> lock get <object-name> <lock-name> <locker-id> ""
rados -p <pool-name> lock list <object-name>
rados -p <pool-name> lock break <object-name> <lock-name> <locker-id>

# Show RADOS df
rados df

# Run a bench test
rados -p <pool-name> bench <seconds> write --no-cleanup
rados -p <pool-name> bench <seconds> seq
rados -p <pool-name> bench <seconds> rand
rados -p <pool-name> cleanup

# Watch cluster events
rados -p <pool-name> watch <object-name>
rados -p <pool-name> notify <object-name> <message>

# Import/export a pool (stop I/O on source for consistent cutover)
rados export [--create] <pool-name> /path/to/dir_or_file
rados import /path/to/dir_or_file <dest-pool>
rados export --workers 8 <pool-name> /path/to/dir
rados import --workers 8 /path/to/dir <dest-pool>

# Copy all objects between pools on the same cluster (simple migration)
rados cppool <src-pool> <dst-pool>    # not valid for all pool types (e.g. some EC)

# Determine which PG an object belongs to
ceph osd map <pool-name> <object-name>

# List objects in an object namespace (not pool namespace — RADOS ns)
rados -p <pool-name> ls -N <namespace>
```

---

## 12. RBD — Block Storage

```bash
# Create an image
rbd create <pool>/<image-name> --size <size>         # e.g. --size 10G

# Pool namespaces (multi-tenant isolation within a pool)
rbd namespace create <pool>/<namespace>
rbd namespace ls <pool>
rbd namespace rm <pool>/<namespace>

# List images
rbd ls <pool>
rbd ls -l <pool>                                     # long listing with size

# Show image info
rbd info <pool>/<image-name>

# Provisioned vs actual usage (sparse images)
rbd du <pool>/<image-name>

# Compare image to snapshot or another image (changed extents)
rbd diff <pool>/<image-name> --from-snap <snap>

# Watchers / locks (who has the image open)
rbd status <pool>/<image-name>
rbd lock ls <pool>/<image-name>
rbd lock add <pool>/<image-name> <lock-id> --shared
rbd lock rm <pool>/<image-name> <lock-id> <locker>

# Resize an image
rbd resize <pool>/<image-name> --size <new-size>
rbd resize <pool>/<image-name> --size <new-size> --allow-shrink

# Remove an image
rbd rm <pool>/<image-name>

# Rename an image
rbd rename <pool>/<old-name> <pool>/<new-name>

# Copy an image
rbd copy <pool>/<image-name> <dest-pool>/<dest-image>

# Move an image
rbd mv <pool>/<image-name> <dest-pool>/<dest-image>

# Map an image to a block device (kernel module)
rbd map <pool>/<image-name>
rbd map <pool>/<image-name> --id <user> --keyfile /etc/ceph/ceph.client.<user>.keyring

# Show mapped devices
rbd showmapped
rbd device list

# Unmap a device
rbd unmap /dev/rbd<n>
rbd unmap <pool>/<image-name>

# RBD snapshots
rbd snap create <pool>/<image-name>@<snap-name>
rbd snap ls <pool>/<image-name>
rbd snap rollback <pool>/<image-name>@<snap-name>
rbd snap rm <pool>/<image-name>@<snap-name>
rbd snap purge <pool>/<image-name>                   # remove all snaps
rbd snap protect <pool>/<image-name>@<snap-name>     # protect before cloning
rbd snap unprotect <pool>/<image-name>@<snap-name>

# Clone an image from a snapshot
rbd clone <pool>/<image-name>@<snap-name> <dest-pool>/<clone-name>

# Flatten a clone (remove parent dependency)
rbd flatten <pool>/<clone-name>

# Show parent of a clone
rbd info <pool>/<clone-name> | grep parent

# List children of a snapshot
rbd children <pool>/<image-name>@<snap-name>

# RBD image features
rbd feature enable  <pool>/<image-name> <feature>
rbd feature disable <pool>/<image-name> <feature>
# Common features: layering, striping, exclusive-lock, object-map,
#                  fast-diff, deep-flatten, journaling, data-pool

# Export/import
rbd export <pool>/<image-name> /path/to/file.raw
rbd export <pool>/<image-name>@<snap> /path/to/snap.raw
rbd import /path/to/file.raw <pool>/<image-name>

# Export diff (for incremental backups)
rbd export-diff <pool>/<image-name>@<snap2> \
  --from-snap <snap1> /path/to/diff.bin
rbd import-diff /path/to/diff.bin <pool>/<image-name>

# Discard unused blocks (sparse / TRIM — frees cluster space)
rbd sparsify <pool>/<image-name>

# Trash (soft delete)
rbd trash mv <pool>/<image-name>
rbd trash ls <pool>
rbd trash restore <pool> <image-id>
rbd trash rm <pool> <image-id>
rbd trash purge <pool>

# Performance stats
rbd perf image stats
rbd perf image stats <pool>/<image-name>
rbd perf image iostat

# RBD mirror (cross-cluster replication)
rbd mirror pool enable <pool> pool
rbd mirror pool enable <pool> image
rbd mirror pool disable <pool>
rbd mirror pool info <pool>
rbd mirror pool status <pool>
rbd mirror pool peer add <pool> <cluster-name>
rbd mirror pool peer remove <pool> <peer-uuid>
rbd mirror image enable <pool>/<image-name> snapshot
rbd mirror image enable <pool>/<image-name> journal
rbd mirror image disable <pool>/<image-name>
rbd mirror image status <pool>/<image-name>
rbd mirror image promote <pool>/<image-name>
rbd mirror image demote <pool>/<image-name>
rbd mirror image resync <pool>/<image-name>

# rbd-nbd (NBD-backed mapping, for older kernels)
rbd-nbd map <pool>/<image-name>
rbd-nbd list
rbd-nbd unmap /dev/nbd<n>

# Benchmark an image
rbd bench --io-type write --io-size 4K --io-threads 16 \
  --io-total 1G <pool>/<image-name>
```

---

## 13. CephFS — Filesystem

```bash
# Create a CephFS volume (recommended: ceph fs volume)
ceph fs volume create <vol-name>

# List CephFS volumes
ceph fs volume ls

# Create CephFS manually (old way)
ceph osd pool create cephfs_data <pg_num>
ceph osd pool create cephfs_metadata <pg_num>
ceph fs new <fs-name> cephfs_metadata cephfs_data

# List filesystems
ceph fs ls
ceph fs dump

# Get filesystem status
ceph fs status
ceph fs status <fs-name>

# Remove a CephFS (fail first if still online)
ceph fs fail <fs-name>
ceph fs rm <fs-name> --yes-i-really-mean-it

# Fail / recover FS (emergency or upgrade)
ceph fs fail <fs-name>
ceph fs set <fs-name> joinable true          # recover after fail

# Volume management (volumes API)
ceph fs volume rm <vol-name> [--yes-i-really-mean-it]

# Show FS parameters
ceph fs get <fs-name>

# Set FS parameters
ceph fs set <fs-name> max_mds <n>            # scale MDS daemons
ceph fs set <fs-name> allow_standby_replay true
ceph fs set <fs-name> standby_count_wanted <n>
ceph fs set <fs-name> down false

# Mount CephFS with kernel client
mount -t ceph <MON_IP>:/ /mnt/cephfs \
  -o name=admin,secret=<KEY>

mount -t ceph <MON_IP>:/ /mnt/cephfs \
  -o name=admin,secretfile=/etc/ceph/ceph.client.admin.keyring

# Mount a specific subdirectory
mount -t ceph <MON_IP>:/path/to/dir /mnt/cephfs \
  -o name=admin,secretfile=/etc/ceph/ceph.client.admin.keyring

# Mount with ceph-fuse (FUSE client)
ceph-fuse /mnt/cephfs
ceph-fuse /mnt/cephfs --id <client-id>
ceph-fuse /mnt/cephfs -o allow_other

# Unmount
fusermount -u /mnt/cephfs
umount /mnt/cephfs

# fstab entry (kernel client)
# <MON_IP>:/ /mnt/cephfs ceph name=admin,secretfile=/etc/ceph/secret.key,_netdev 0 0

# CephFS subvolumes (multi-tenancy)
ceph fs subvolume create <fs-name> <subvol-name> [--group_name <group>]
ceph fs subvolume ls <fs-name>
ceph fs subvolume info <fs-name> <subvol-name>
ceph fs subvolume getpath <fs-name> <subvol-name>
ceph fs subvolume rm <fs-name> <subvol-name>
ceph fs subvolume resize <fs-name> <subvol-name> <size>

# Subvolume groups
ceph fs subvolumegroup create <fs-name> <group-name>
ceph fs subvolumegroup ls <fs-name>
ceph fs subvolumegroup rm <fs-name> <group-name>

# CephFS snapshots (via subvolume)
ceph fs subvolume snapshot create <fs-name> <subvol-name> <snap-name>
ceph fs subvolume snapshot ls <fs-name> <subvol-name>
ceph fs subvolume snapshot rm <fs-name> <subvol-name> <snap-name>

# Client sessions
ceph tell mds.<id> session ls
ceph tell mds.<id> session evict <client-id>

# CephFS quotas (via extended attributes)
setfattr -n ceph.quota.max_bytes -v <bytes> /mnt/cephfs/<dir>
setfattr -n ceph.quota.max_files -v <n> /mnt/cephfs/<dir>
getfattr -n ceph.quota.max_bytes /mnt/cephfs/<dir>
getfattr -n ceph.quota.max_files /mnt/cephfs/<dir>

# Show CephFS layout
getfattr -n ceph.file.layout /mnt/cephfs/<file>
getfattr -n ceph.dir.layout /mnt/cephfs/<dir>
setfattr -n ceph.dir.layout.stripe_unit -v <bytes> /mnt/cephfs/<dir>
setfattr -n ceph.dir.layout.stripe_count -v <n> /mnt/cephfs/<dir>
setfattr -n ceph.dir.layout.pool -v <pool-name> /mnt/cephfs/<dir>
```

---

## 14. RGW — Object Gateway (S3/Swift)

```bash
# Deploy RGW via cephadm (realm/zone must exist in RGW period, or use defaults)
ceph orch apply rgw --svc_id=<service-id> --realm=<realm> --zone=<zone> \
  [--zonegroup=<zg>] --placement="<hosts-or-count>"
ceph orch apply rgw --svc_id=myrgw --realm=default --zone=default --placement="count:2"

# NFS-Ganesha via orchestrator (see also §15 `ceph nfs`; pick one style per release)
ceph orch apply nfs --svc_id=<id> --placement="<hosts>"

# List RGW daemons
ceph orch ls --service-type rgw

# RGW realm/zone/zonegroup management
radosgw-admin realm create --rgw-realm=<realm> --default
radosgw-admin realm list
radosgw-admin realm get --rgw-realm=<realm>
radosgw-admin realm delete --rgw-realm=<realm>

radosgw-admin zonegroup create --rgw-zonegroup=<zonegroup> \
  --rgw-realm=<realm> --master --default
radosgw-admin zonegroup list
radosgw-admin zonegroup get --rgw-zonegroup=<zonegroup>
radosgw-admin zonegroup modify --rgw-zonegroup=<zonegroup>

radosgw-admin zone create --rgw-zonegroup=<zonegroup> \
  --rgw-zone=<zone> --master --default \
  --endpoints=http://<host>:<port>
radosgw-admin zone list
radosgw-admin zone get --rgw-zone=<zone>
radosgw-admin zone modify --rgw-zone=<zone>
radosgw-admin zone delete --rgw-zone=<zone>

radosgw-admin period update --commit

# Placement targets (storage classes / tiers — zone-level)
radosgw-admin zone placement list --rgw-zone=<zone>
radosgw-admin zone placement add --rgw-zone=<zone> --placement-id=<id> \
  --data-pool=<pool> [--index-pool=<pool>] [--extra-pool=<pool>]
radosgw-admin zone placement rm --rgw-zone=<zone> --placement-id=<id>

# User management
radosgw-admin user create \
  --uid=<user-id> \
  --display-name="<display name>" \
  --email=<email>

radosgw-admin user create \
  --uid=<user-id> \
  --display-name="<display name>" \
  --access-key=<key> \
  --secret=<secret>

radosgw-admin user list
radosgw-admin user info --uid=<user-id>
radosgw-admin user modify --uid=<user-id> --max-buckets=<n>
radosgw-admin user suspend --uid=<user-id>
radosgw-admin user enable --uid=<user-id>
radosgw-admin user rm --uid=<user-id>
radosgw-admin user rm --uid=<user-id> --purge-data

# Sub-users
radosgw-admin subuser create --uid=<user-id> --subuser=<user>:<subuser>
radosgw-admin subuser modify --uid=<user-id> --subuser=<user>:<subuser>
radosgw-admin subuser rm --uid=<user-id> --subuser=<user>:<subuser>

# Access keys
radosgw-admin key create --uid=<user-id> --key-type=s3
radosgw-admin key rm --uid=<user-id> --access-key=<key>
radosgw-admin key create --uid=<user-id> --key-type=swift \
  --subuser=<user>:<subuser>

# Bucket management
radosgw-admin bucket list
radosgw-admin bucket list --uid=<user-id>
radosgw-admin bucket stats
radosgw-admin bucket stats --bucket=<bucket>
radosgw-admin bucket check --bucket=<bucket>
radosgw-admin bucket link --uid=<user-id> --bucket=<bucket>
radosgw-admin bucket unlink --uid=<user-id> --bucket=<bucket>
radosgw-admin bucket rm --bucket=<bucket>
radosgw-admin bucket rm --bucket=<bucket> --purge-objects

# Quota management
radosgw-admin quota set --uid=<user-id> --quota-scope=user \
  --max-size=<size> --max-objects=<n>
radosgw-admin quota enable --uid=<user-id> --quota-scope=user
radosgw-admin quota disable --uid=<user-id> --quota-scope=user
radosgw-admin quota set --bucket=<bucket> --quota-scope=bucket \
  --max-size=<size> --max-objects=<n>
radosgw-admin quota enable --bucket=<bucket> --quota-scope=bucket

# Usage/billing logs
radosgw-admin usage show --uid=<user-id>
radosgw-admin usage show --uid=<user-id> --start-date=<date> --end-date=<date>
radosgw-admin usage trim --uid=<user-id>

# Multisite sync
radosgw-admin sync status
radosgw-admin data sync status --source-zone=<zone>
radosgw-admin metadata sync status
radosgw-admin sync error list
radosgw-admin sync error trim

# Garbage collection
radosgw-admin gc list
radosgw-admin gc process
radosgw-admin gc list --include-all

# Object expire
radosgw-admin lc list
radosgw-admin lc process
radosgw-admin lc get --bucket=<bucket>

# Orphan objects
radosgw-admin orphans find --pool=<pool> --job-id=<id>
radosgw-admin orphans finish --job-id=<id>
radosgw-admin orphans list-jobs

# RGW admin REST API
curl -s http://<rgw-host>:<port>/admin/usage?uid=<uid>&format=json
```

---

## 15. NFS Gateway

```bash
# Deploy NFS cluster via cephadm
ceph nfs cluster create <cluster-id> --placement="<host>"

# List NFS clusters
ceph nfs cluster ls
ceph nfs cluster info <cluster-id>

# Delete NFS cluster
ceph nfs cluster delete <cluster-id>

# Create NFS export for CephFS
ceph nfs export create cephfs \
  --cluster-id <cluster-id> \
  --pseudo-path /export \
  --fsname <fs-name> \
  --path /

# Create NFS export for RGW
ceph nfs export create rgw \
  --cluster-id <cluster-id> \
  --pseudo-path /rgw \
  --bucket <bucket>

# List exports
ceph nfs export ls <cluster-id>
ceph nfs export info <cluster-id> <pseudo-path>

# Apply an export from YAML
ceph nfs export apply <cluster-id> -i export.yaml

# Delete an export
ceph nfs export delete <cluster-id> <pseudo-path>

# Get NFS-Ganesha config
ceph nfs cluster config get <cluster-id>

# Mount the NFS export
mount -t nfs -o vers=4.1 <nfs-host>:/<pseudo-path> /mnt/nfs
```

---

## 16. iSCSI Gateway

```bash
# Deploy iSCSI gateway
ceph orch apply iscsi --pool=<pool> --api_user=<user> --api_password=<pw> \
  [--trusted_ip_list=<cidr>] \
  --placement="host1,host2"

# List iSCSI daemons
ceph orch ls --service-type iscsi

# Use gwcli to manage iSCSI targets
gwcli                                    # interactive CLI

# Create a target
gwcli /iscsi-targets create iqn.2003-01.com.redhat.iscsi-gw:iscsi-igw

# Add a gateway
gwcli /iscsi-targets/iqn.2003-01.com.redhat.iscsi-gw:iscsi-igw/gateways \
  create <gateway-host> <IP>

# Add a disk
gwcli /disks create pool=<pool> image=<image> size=<size>

# Map disk to target
gwcli /iscsi-targets/<iqn>/disks add <pool>/<image>

# Create a client
gwcli /iscsi-targets/<iqn>/hosts create iqn.1994-05.com.redhat:<client>
```

---

## 17. Auth & Keyring Management

```bash
# List all auth entities
ceph auth ls

# Get a specific entity's key
ceph auth get client.admin
ceph auth get osd.<id>
ceph auth get mds.<id>

# Print only the key value
ceph auth print-key client.admin

# Create a new key with capabilities
ceph auth get-or-create client.<name> \
  mon 'profile rbd' \
  osd 'profile rbd pool=<pool>' \
  mgr 'profile rbd pool=<pool>'

# Create a CephFS client key
ceph auth get-or-create client.<name> \
  mon 'allow r' \
  mds 'allow rw path=/<dir>' \
  osd 'allow rw pool=<data-pool>'

# Create a read-only key
ceph auth get-or-create client.readonly \
  mon 'allow r' \
  osd 'allow r'

# Modify capabilities
ceph auth caps client.<name> \
  mon 'allow r' \
  osd 'allow rw pool=<pool>'

# Export a keyring
ceph auth get client.<name> -o /etc/ceph/ceph.client.<name>.keyring

# Import a keyring
ceph auth import -i /etc/ceph/ceph.client.<name>.keyring

# Delete an entity
ceph auth del client.<name>

# Generate a new key (rotate)
ceph auth rotate client.<name>

# Create a keyring file manually
ceph-authtool --create-keyring /tmp/ceph.client.test.keyring \
  --gen-key -n client.test

# Add caps to a keyring file
ceph-authtool /tmp/ceph.client.test.keyring \
  -n client.test \
  --cap mon 'allow r' \
  --cap osd 'allow rw pool=testpool'

# Print a keyring file
ceph-authtool --print-key /etc/ceph/ceph.client.admin.keyring
```

---

## 18. MDS — Metadata Server

```bash
# List MDS daemons
ceph mds stat
ceph fs dump | grep mds

# Deploy MDS
ceph orch apply mds <fs-name> --placement="count:2"
ceph orch daemon add mds <fs-name> <hostname>

# Show active/standby MDS
ceph fs status <fs-name>

# Fail an MDS (force failover)
ceph mds fail <mds-name>

# Set max active MDS
ceph fs set <fs-name> max_mds <n>

# Pin a directory to a specific rank (MDS affinity)
setfattr -n ceph.dir.pin -v <rank> /mnt/cephfs/<dir>
setfattr -n ceph.dir.pin.distributed -v 1 /mnt/cephfs/<dir>
setfattr -n ceph.dir.pin.random -v 0.5 /mnt/cephfs/<dir>

# Scrub the filesystem
ceph tell mds.<rank> scrub start / recursive
ceph tell mds.<rank> scrub start /path recursive repair
ceph tell mds.<rank> scrub status
ceph tell mds.<rank> scrub abort

# MDS cache settings
ceph config set mds mds_cache_memory_limit <bytes>
ceph config get mds mds_cache_memory_limit

ceph config set mds mds_cache_trim_threshold <n>
ceph config get mds mds_cache_trim_threshold

# Evict all clients (emergency)
ceph tell mds.<rank> evict_clients {}

# Export an MDS subtree to another rank
ceph tell mds.<src-rank> export_dir /path <dest-rank>

# Dump MDS session info
ceph daemon mds.<name> session ls

# Show MDS performance counters
ceph daemon mds.<name> perf dump
```

---

## 19. PG (Placement Group) Management

```bash
# Show PG summary
ceph pg stat

# List all PGs
ceph pg ls

# List PGs in a specific state
ceph pg ls-by-pool <pool-name>
ceph pg ls active+degraded
ceph pg ls stale
ceph pg ls unclean

# Show PG info
ceph pg <pg-id> query

# Show PG map
ceph pg map <pg-id>

# Force recovery of a stuck PG
ceph pg force-recovery <pg-id>
ceph pg force-backfill <pg-id>
ceph pg cancel-force-recovery <pg-id>
ceph pg cancel-force-backfill <pg-id>

# Repair a PG
ceph pg repair <pg-id>

# Scrub a PG
ceph pg scrub <pg-id>
ceph pg deep-scrub <pg-id>

# Show PG autoscaler status
ceph osd pool autoscale-status

# Show PG distribution
ceph pg dump pgs_brief

# Show PGs on a specific OSD
ceph pg ls-by-osd <osd-id>

# Show PGs on a specific primary OSD
ceph pg ls-by-primary <osd-id>

# Dump all PG stats
ceph pg dump

# Dump PG stats in JSON
ceph pg dump --format json-pretty

# Show stuck PGs
ceph pg dump_stuck unclean
ceph pg dump_stuck inactive
ceph pg dump_stuck stale

# Merge/split PGs (Nautilus+)
ceph osd pool set <pool> pg_num_min <n>
ceph osd pool set <pool> pg_num <new-n>         # triggers split/merge
```

---

## 20. Monitoring & Alerting

```bash
# Enable Prometheus module
ceph mgr module enable prometheus

# Show Prometheus endpoint
ceph mgr services | grep prometheus
# Default: http://<mgr-host>:9283/metrics

# Enable the dashboard
ceph mgr module enable dashboard
ceph dashboard create-self-signed-cert
ceph dashboard set-login-credentials <user> <password>

# RBAC user (instead of only bootstrap admin)
ceph dashboard ac-user-create <user> -i <pwfile> administrator
ceph dashboard ac-user-show <user>
ceph dashboard ac-user-delete <user>

# Get dashboard URL
ceph mgr services

# Enable Grafana integration
ceph dashboard set-grafana-api-url http://<grafana-host>:3000
ceph dashboard set-grafana-api-ssl-verify false
ceph dashboard set-alertmanager-api-host http://<alertmanager>:9093

# Deploy monitoring stack via cephadm
ceph orch apply prometheus
ceph orch apply grafana
ceph orch apply alertmanager
ceph orch apply node-exporter

# List monitoring services
ceph orch ls --service-type prometheus
ceph orch ls --service-type grafana

# Check Prometheus targets
curl -s http://<mgr-host>:9283/metrics | head -50

# Telegraf integration (for InfluxDB)
ceph mgr module enable telegraf
ceph config set mgr mgr/telegraf/address udp://localhost:8092
ceph config set mgr mgr/telegraf/interval 15

# Cluster log
ceph log last <n>
ceph log last <n> --channel cluster
ceph log last <n> --channel audit

# Enable cluster-wide log to file
ceph config set global log_to_file true
ceph config set global log_file /var/log/ceph/ceph.log
```

---

## 21. Performance & Benchmarking

```bash
# RADOS benchmark (write)
rados -p <pool> bench 60 write --no-cleanup -b 4096

# RADOS benchmark (sequential read)
rados -p <pool> bench 60 seq

# RADOS benchmark (random read)
rados -p <pool> bench 60 rand

# Cleanup bench objects
rados -p <pool> cleanup

# RBD benchmark
rbd bench --io-type write --io-size 4K --io-threads 16 \
  --io-total 1G <pool>/<image>

rbd bench --io-type read --io-size 4K --io-threads 16 \
  --io-total 1G <pool>/<image>

# fio via rbd engine
fio --name=test --ioengine=rbd --pool=<pool> --rbdname=<image> \
  --rw=randwrite --bs=4k --iodepth=32 --runtime=60 --time_based

# Show live OSD throughput
ceph osd perf

# Enable/disable mclock QoS scheduler
ceph config set osd osd_op_queue mclock_scheduler
ceph config set osd osd_op_queue wpq               # weighted priority queue

# Tune BlueStore cache
ceph config set osd bluestore_cache_size_ssd <bytes>
ceph config set osd bluestore_cache_size_hdd <bytes>
ceph config set osd bluestore_cache_meta_ratio 0.4
ceph config set osd bluestore_cache_kv_ratio 0.4

```

---

## 22. Scrubbing & Repair

```bash
# Manual scrub
ceph osd scrub <osd-id>
ceph pg scrub <pg-id>

# Deep scrub
ceph osd deep-scrub <osd-id>
ceph pg deep-scrub <pg-id>

# Repair a PG (after scrub found errors)
ceph pg repair <pg-id>

# Schedule/configure scrubbing
ceph config set osd osd_scrub_begin_hour <0-23>
ceph config set osd osd_scrub_end_hour <0-23>
ceph config set osd osd_scrub_min_interval <seconds>
ceph config set osd osd_scrub_max_interval <seconds>
ceph config set osd osd_deep_scrub_interval <seconds>

# Disable scrubbing cluster-wide
ceph osd set noscrub
ceph osd set nodeep-scrub

# Re-enable scrubbing
ceph osd unset noscrub
ceph osd unset nodeep-scrub

# Show scrub errors
ceph health detail | grep scrub

# List inconsistent objects
rados list-inconsistent-pg <pool>
rados list-inconsistent-obj <pg-id>
rados list-inconsistent-snapset <pg-id>

# Export a broken PG (for recovery)
ceph-objectstore-tool --data-path /var/lib/ceph/osd/ceph-<id>/ \
  --pgid <pg-id> --op export --file /tmp/<pg-id>.bin

# Import a PG
ceph-objectstore-tool --data-path /var/lib/ceph/osd/ceph-<id>/ \
  --op import --file /tmp/<pg-id>.bin

# Remove a broken PG (DANGEROUS)
ceph-objectstore-tool --data-path /var/lib/ceph/osd/ceph-<id>/ \
  --pgid <pg-id> --op remove
```

---

## 23. Snapshots & Clones

```bash
# RBD snapshots (see RBD section for full list)
rbd snap create <pool>/<image>@<snap>
rbd snap ls <pool>/<image>
rbd snap rollback <pool>/<image>@<snap>
rbd snap rm <pool>/<image>@<snap>
rbd snap purge <pool>/<image>

# Group snapshots (crash-consistent, multi-image)
rbd group create <pool>/<group-name>
rbd group ls <pool>
rbd group image add <pool>/<group-name> <pool>/<image>
rbd group image rm <pool>/<group-name> <pool>/<image>
rbd group image list <pool>/<group-name>
rbd group snap create <pool>/<group-name>@<snap>
rbd group snap ls <pool>/<group-name>
rbd group snap rollback <pool>/<group-name>@<snap>
rbd group snap rm <pool>/<group-name>@<snap>
rbd group rm <pool>/<group-name>

# CephFS snapshots
mkdir /mnt/cephfs/<dir>/.snap/<snap-name>       # create
ls /mnt/cephfs/<dir>/.snap/                     # list
rmdir /mnt/cephfs/<dir>/.snap/<snap-name>        # delete

# CephFS scheduled snapshots (mgr module: ceph mgr module enable snap_schedule)
ceph fs snap-schedule add / 1h                   # every hour
ceph fs snap-schedule add /mydir 1d --fs <fs>
ceph fs snap-schedule activate / 1h
ceph fs snap-schedule deactivate / 1h
ceph fs snap-schedule status /
ceph fs snap-schedule list /
ceph fs snap-schedule remove / 1h
ceph fs snap-schedule retention add / h 24       # keep 24 hourly
ceph fs snap-schedule retention add / d 7        # keep 7 daily
ceph fs snap-schedule retention remove / h 24

# Pool-level snapshots (RADOS)
ceph osd pool mksnap <pool> <snap-name>
ceph osd pool ls detail | grep snap
ceph osd pool rmsnap <pool> <snap-name>

# Clone an RBD image from a snapshot
rbd snap protect <pool>/<image>@<snap>
rbd clone <pool>/<image>@<snap> <dest-pool>/<clone>
rbd flatten <dest-pool>/<clone>
```

---

## 24. Quota Management

```bash
# Pool-level quotas
ceph osd pool set-quota <pool> max_bytes <bytes>
ceph osd pool set-quota <pool> max_objects <n>
ceph osd pool get-quota <pool>

# RGW user quotas
radosgw-admin quota set --uid=<uid> --quota-scope=user \
  --max-size=10G --max-objects=100000
radosgw-admin quota enable  --uid=<uid> --quota-scope=user
radosgw-admin quota disable --uid=<uid> --quota-scope=user
radosgw-admin quota set --uid=<uid> --quota-scope=bucket \
  --max-size=5G --max-objects=50000

# RGW global quotas
radosgw-admin global quota set --quota-scope=user \
  --max-size=50G --max-objects=1000000
radosgw-admin global quota enable  --quota-scope=user
radosgw-admin global quota disable --quota-scope=user
radosgw-admin global quota get

# CephFS directory quotas (via xattrs)
setfattr -n ceph.quota.max_bytes   -v <bytes> /mnt/cephfs/<dir>
setfattr -n ceph.quota.max_files   -v <n>     /mnt/cephfs/<dir>
getfattr -n ceph.quota.max_bytes   /mnt/cephfs/<dir>
getfattr -n ceph.quota.max_files   /mnt/cephfs/<dir>
# Remove quota (set to 0)
setfattr -n ceph.quota.max_bytes   -v 0 /mnt/cephfs/<dir>
setfattr -n ceph.quota.max_files   -v 0 /mnt/cephfs/<dir>
```

---

## 25. Configuration Management

```bash
# Show all runtime config
ceph config dump

# Show config for a specific entity
ceph config get osd
ceph config get osd.<id>
ceph config get mon
ceph config get mds
ceph config get mgr
ceph config get client

# Set a config value
ceph config set global <key> <value>
ceph config set osd <key> <value>
ceph config set osd.<id> <key> <value>
ceph config set mon <key> <value>
ceph config set mds <key> <value>
ceph config set client <key> <value>

# Remove a config value (revert to default)
ceph config rm global <key>
ceph config rm osd <key>

# Show config with sources
ceph config show osd.<id>
ceph config show-with-defaults osd.<id>

# Inject a config at runtime (non-persistent)
ceph tell osd.<id> injectargs '--<key>=<value>'
ceph tell osd.* injectargs '--osd_max_backfills=2'
ceph tell mon.* injectargs '--mon_allow_pool_delete=true'

# Get a specific value at runtime
ceph daemon osd.<id> config get <key>

# Show ceph.conf
cat /etc/ceph/ceph.conf

# Validate a ceph.conf
ceph-conf --show-config-value <key>

# Common tuning parameters
ceph config set osd osd_max_backfills 1
ceph config set osd osd_recovery_max_active 3
ceph config set osd osd_recovery_op_priority 3
ceph config set osd osd_client_op_priority 63
ceph config set osd osd_backfill_scan_min 16
ceph config set osd osd_backfill_scan_max 512
ceph config set osd osd_heartbeat_interval 6
ceph config set osd osd_heartbeat_grace 20

# Config assimilation (from ceph.conf into mon store)
ceph config assimilate-conf -i /etc/ceph/ceph.conf
```

---

## 26. Orchestrator & Service Specs

```bash
# List all services
ceph orch ls
ceph orch ls --service-type osd
ceph orch ls --format yaml
ceph orch ls --export               # emit runnable YAML specs (backup / GitOps)
ceph orch ls --refresh              # refresh from hosts (slower, use if state looks stale)

# List all daemons
ceph orch ps
ceph orch ps --daemon-type osd
ceph orch ps <hostname>

# Apply a service spec from YAML
ceph orch apply -i service-spec.yaml

# Redeploy a service
ceph orch redeploy <service-name>

# Restart a daemon
ceph orch daemon restart <daemon-name>     # e.g. osd.0, mon.host1

# Stop/start a daemon
ceph orch daemon stop  <daemon-name>
ceph orch daemon start <daemon-name>

# Remove a daemon
ceph orch daemon rm <daemon-name>

# Remove a service entirely
ceph orch rm <service-name>

# Pause/resume the orchestrator
ceph orch pause
ceph orch resume

# Check orchestrator status
ceph orch status

# Example OSD service spec (YAML)
cat <<EOF > osd-spec.yaml
service_type: osd
service_id: default
placement:
  host_pattern: '*'
spec:
  data_devices:
    all: true
  db_devices:
    model: <SSD_MODEL>
  wal_devices:
    model: <NVME_MODEL>
EOF
ceph orch apply -i osd-spec.yaml

# Example RGW service spec
cat <<EOF > rgw-spec.yaml
service_type: rgw
service_id: myrgw
placement:
  count: 2
spec:
  rgw_realm: myrealm
  rgw_zone: myzone
  ssl: true
  rgw_frontend_ssl_certificate: |
    -----BEGIN CERTIFICATE-----
    ...
EOF
ceph orch apply -i rgw-spec.yaml

# Preview what the orchestrator would do
ceph orch apply osd --all-available-devices --dry-run

# SSH key management for cephadm (bootstrap / rotation)
ceph cephadm generate-key
ceph cephadm get-pub-key
ceph cephadm set-pub-key -i /root/.ssh/id_rsa.pub
ceph cephadm set-priv-key -i /root/.ssh/id_rsa

# Non-root SSH user for cephadm agents
ceph config set mgr mgr/cephadm/ssh_user <user>

# Minimal ceph.conf (MON-backed config + minimal sections)
ceph config generate-minimal-conf
```

---

## 27. Upgrade & Maintenance

```bash
# Check current versions
ceph versions
ceph version

# Upgrade via cephadm (check = versions vs target image / release)
ceph orch upgrade check <container-image> <ceph-version>   # e.g. quay.io/ceph/ceph:v18 18.2.0
ceph orch upgrade start --ceph-version <x.y.z>
ceph orch upgrade start --image quay.io/ceph/ceph:v18.2.0

# Monitor upgrade progress
ceph orch upgrade status
ceph orch upgrade ls

# Pause/resume upgrade
ceph orch upgrade pause
ceph orch upgrade resume

# Stop upgrade
ceph orch upgrade stop

# Pre-upgrade checks
ceph health detail
ceph versions
ceph osd ok-to-stop --ids=<osd-id> [<osd-id> ...]
ceph mds ok-to-stop --ids=<mds-id-or-name>
ceph mon ok-to-stop --ids=<mon-name>
ceph orch host ok-to-stop <hostname>

# Set cluster flags before maintenance
ceph osd set noout
ceph osd set norebalance
ceph osd set nobackfill

# Unset after maintenance
ceph osd unset noout
ceph osd unset norebalance
ceph osd unset nobackfill

# Rolling restart workflow
# 1. Check health: ceph -s
# 2. Set noout
# 3. Restart one OSD at a time
ceph orch daemon restart osd.<id>
# 4. Wait for health to recover
# 5. Repeat for all OSDs, then MONs, then MGRs

# Backup cluster state
ceph mon getmap -o /backup/monmap.bin
ceph osd getmap -o /backup/osdmap.bin
ceph osd getcrushmap -o /backup/crushmap.bin
ceph auth export -o /backup/ceph.auth.export
ceph config dump > /backup/ceph-config.txt
```

---

## 28. Disaster Recovery & Troubleshooting

```bash
# Cluster is stuck / not forming quorum
# Force a single mon to form quorum (DANGEROUS)
ceph-mon -i <hostname> --inject-monmap /tmp/monmap --keyring /etc/ceph/ceph.mon.keyring

# Rebuild the monitor store
ceph-monstore-tool /var/lib/ceph/mon/ceph-<hostname>/ rebuild

# Recover from lost monmap
ceph-mon --extract-monmap /tmp/monmap.bin
monmaptool --print /tmp/monmap.bin

# Recover from total OSD failure (mark lost)
ceph osd lost <osd-id> --yes-i-really-mean-it

# Force import of a pg epoch
ceph-objectstore-tool --data-path /var/lib/ceph/osd/ceph-<id> \
  --pgid <pg-id> --op trim-pg-log

# Find and fix unfound objects
ceph pg <pg-id> query | grep unfound
ceph pg <pg-id> mark_unfound_lost revert
ceph pg <pg-id> mark_unfound_lost delete

# Force a specific PG to be active
ceph pg force-recovery <pg-id>

# Inspect daemon logs
sudo journalctl -u "ceph*"

# Grep for slow ops in logs
journalctl -u "ceph*" | grep "slow request"

# Inspect BlueStore corruption
ceph-bluestore-tool show-label --dev /dev/<device>
ceph-bluestore-tool fsck --path /var/lib/ceph/osd/ceph-<id>
ceph-bluestore-tool repair --path /var/lib/ceph/osd/ceph-<id>
ceph-bluestore-tool prime-osd-dir --dev /dev/<device> \
  --path /var/lib/ceph/osd/ceph-<id>

# Check and repair BlueFS
ceph-bluestore-tool bluefs-bdev-sizes --path /var/lib/ceph/osd/ceph-<id>
ceph-bluestore-tool bluefs-bdev-expand --path /var/lib/ceph/osd/ceph-<id>

# Send commands to a subsystem (daemon or wildcard)
ceph tell mon.* version
ceph tell osd.0 bench 4096 10
ceph tell mds.<name> respawn

# Device health (SMART / predictions — mgr module + orchestrator)
ceph device ls
ceph device info <device-id>
ceph device predict-health <device-id>
ceph device light --enable=on --devid=<device-id> --light_type=ident   # enclosure LED (hardware permitting)
ceph device light --enable=off --devid=<device-id>

# Check for clock skew
ceph health detail | grep clock

# Reset a crashed daemon
ceph crash archive-all

# Emergency: stop all IO
ceph osd set pause
# Resume IO
ceph osd unset pause

# Check if any PGs are blocked
ceph pg ls | grep -v active+clean
```

---

## 29. Client Setup & Integration

```bash
# Install Ceph client packages
# Debian/Ubuntu
apt install ceph-common

# RHEL/CentOS/Rocky
dnf install ceph-common

# Copy config and keyring to client
scp root@<ceph-host>:/etc/ceph/ceph.conf   /etc/ceph/ceph.conf
scp root@<ceph-host>:/etc/ceph/ceph.client.admin.keyring \
  /etc/ceph/ceph.client.admin.keyring

# Create a restricted client key for an application
ceph auth get-or-create client.myapp \
  mon 'allow r' \
  osd 'allow rwx pool=mypool' \
  -o /etc/ceph/ceph.client.myapp.keyring

# Verify connectivity from client
ceph -s --id myapp --keyring /etc/ceph/ceph.client.myapp.keyring

# Mount RBD block device on client
rbd map mypool/myimage --id myapp \
  --keyring /etc/ceph/ceph.client.myapp.keyring
mkfs.ext4 /dev/rbd0
mount /dev/rbd0 /mnt/rbd

# Auto-map RBD at boot (rbdmap)
echo "mypool/myimage id=myapp,keyring=/etc/ceph/ceph.client.myapp.keyring" \
  >> /etc/ceph/rbdmap
systemctl enable rbdmap
systemctl start rbdmap

# Mount CephFS on client (kernel)
mount -t ceph <MON1>,<MON2>,<MON3>:/ /mnt/cephfs \
  -o name=admin,secretfile=/etc/ceph/ceph.client.admin.keyring

# Mount CephFS via /etc/fstab
# <MON_IP>:/  /mnt/cephfs  ceph  name=admin,secretfile=/etc/ceph/secret,_netdev,noatime  0  0

# Mount CephFS with FUSE on client
ceph-fuse /mnt/cephfs --id admin

# S3 client (AWS CLI) for Ceph RGW
aws configure                           # set access key, secret, region
aws --endpoint-url http://<rgw-host>:7480 s3 ls
aws --endpoint-url http://<rgw-host>:7480 s3 mb s3://mybucket
aws --endpoint-url http://<rgw-host>:7480 s3 cp file.txt s3://mybucket/
aws --endpoint-url http://<rgw-host>:7480 s3 sync /local/path s3://mybucket/

# s3cmd for Ceph RGW
s3cmd --configure                      # set host_base, host_bucket, keys
s3cmd ls
s3cmd mb s3://mybucket
s3cmd put file.txt s3://mybucket/
s3cmd get s3://mybucket/file.txt .

# rclone for Ceph RGW
rclone config                          # add S3-compatible remote
rclone ls ceph:mybucket
rclone sync /local/path ceph:mybucket

# Verify RBD kernel module
lsmod | grep rbd
modprobe rbd
```

---

## Quick Reference: Common Flags & States

| Flag | Description |
|------|-------------|
| `noout` | Prevent OSDs from being marked out on failure |
| `norebalance` | Stop CRUSH rebalancing |
| `nobackfill` | Stop backfill operations |
| `norecover` | Stop recovery operations |
| `noscrub` | Disable scheduled scrubbing |
| `nodeep-scrub` | Disable scheduled deep scrubbing |
| `pause` | Pause all client I/O |
| `full` | Cluster is full, read-only mode |
| `nearfull` | Cluster is near full (warning) |

| PG State | Meaning |
|----------|---------|
| `active+clean` | Healthy — all replicas present and up-to-date |
| `active+degraded` | Some replicas missing but still serving I/O |
| `active+recovering` | Recovering lost replicas in background |
| `active+backfilling` | Backfilling new OSD with data |
| `peering` | OSDs negotiating state (transient) |
| `stale` | Primary OSD hasn't reported in |
| `undersized` | Fewer replicas than `size` |
| `inconsistent` | Scrub found data mismatch between replicas |
| `incomplete` | Not enough replicas to determine state |
| `remapped` | PG mapped to different OSDs than expected |

---

## Tell / admin socket quick reference

```bash
# Cluster-wide or wildcard tells
ceph tell osd.* version
ceph tell mon.* config set debug_mon 10
```

---

## Telemetry

> Optional; disable if your policy forbids usage reporting.

```bash
ceph mgr module enable telemetry    # if not already on
ceph telemetry on
ceph telemetry off
ceph telemetry status
ceph telemetry show               # preview what would be sent
```
---