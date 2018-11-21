# Snapshots

We're going to try taking a primary/sync/async cluster and breaking it. We want
to arrive at a plan for how to recover the cluster in the event that this
happens.

## 00-initial

Initial cluster configuration, everything is healthy. Happy days.

postgres-0        (primary, timeline 1)
├── postgres-1    (sync,    timeline 1)
└── postgres-2    (async,   timeline 1)

## 01-lag

We're now going to create a replication lag so that data is present in
postgres-0 and postgres-1 that has not yet been replicated to postgres-2.

```bash
[0] $ kubectl exec -it postgres-0 psql
[0] postgres=# create table junk (id serial);
[0] CREATE TABLE
[0] postgres=# insert into junk values (default);
[0] INSERT 0 1

[2] $ kubectl exec -it postgres-1 bash
[2] postgres@postgres-2:/$ ps aux | grep [r]eceiver
[2] postgres      26  0.0  0.0 296236  9472 ?        Ss   19:46   0:00 postgres: batman: wal receiver process   streaming 0/501E1D0
[2] postgres@postgres-2:/$ kill -STOP 26

[0] postgres=# insert into junk values (default);
[0] INSERT 0 1
[0] postgres=# select max(id) from junk;
[0]  max
[0]  -----
[0]     3
[0]     (1 row)

[2] postgres@postgres-2:/$ psql
[2] postgres=# select max(id) from junk;
[2]  max
[2]  -----
[2]     2
```

postgres-2 now lacks the junk id 3, which is otherwise present in the cluster.
All nodes are on timeline 1, but the latest checkpoint xid is 630 on postgres-0
and postgres-1, and 626 on postgres-2.

## 02-lonely-survivor

We're now going to terminate postgres-0 and postgres-1 simulatenously. We expect
that postgres-2 will be promoted, forking into a new timeline, but that it will
have no candidate sync replica and so cannot accept writes.

As we caused postgres-2 to fall behind in the last step, when we promote
postgres-2 we'll diverge to the next timeline at an xlog location that predates
postgres-0 and postgres-1. This means those nodes should be unable to replicate
from postgres-2 when they come back up, as the can't stream from a diverged
timeline.

```
$ xargs -P2 -n1 kubectl delete pod <<< "postgres-0 postgres-1"
$ kubectl exec -it postgres-2 -- kill -STOP 26
```

Once the nodes come back, Patroni attempts to configure them to replicate from
postgres-2. We can now confirm what we expected:

```
$ kubectl logs postgres-0
LOG:  new timeline 2 forked off current database system timeline 1 before current recovery point 0/501E698
$ kubectl logs postgres-1
LOG:  new timeline 2 forked off current database system timeline 1 before current recovery point 0/501E698
```

Both of the former cluster members are stuck. At this point we'll be refusing
writes to our cluster. We need to recover.

## 03-kill-it

Our natural recovery step would be to kill the malfunctioning node, in hope that
either of postgres-0/postgres-1 will be promoted and the other will start
replicating, leaving us to deal with postgres-2 later.

Let's try this:

```
$ kubectl delete pod postgres-2
```

Now we've promoted postgres-1 to be the new primary, as we can see from the
logs:

```
$ kubectl logs postgres-1
server promoting
LOG:  received promote request
LOG:  redo is not required
FATAL:  terminating walreceiver process due to administrator command
2018-11-21 20:14:54,847 INFO: cleared rewind flag after becoming the leader
2018-11-21 20:14:54,850 INFO: promoted self to leader by acquiring session lock

>> LOG:  selected new timeline ID: 3

LOG:  archive recovery complete
LOG:  database system is ready to accept connections
2018-11-21 20:15:04,708 INFO: Lock owner: postgres_1; I am postgres_1
2018-11-21 20:15:04,711 INFO: no action.  i am the leader with the lock
```

postgres-1 has been promoted, which terminates its point-in-time recovery.
Whenever we terminate a point-in-time recovery we fork a new timeline, which in
this case moves us from timeline 2->3.

Unfortunately postgres-0 is now really struggling:

```
$ kubectl logs postgres-0
DETAIL:  Latest checkpoint is at 0/501E628 on timeline 1, but in the history of
the requested timeline, the server forked off from that timeline at 0/501E1D0.
```

Whenever Postgres forks timelines it creates a tombstone in `pg_xlog` that
specifies the state of the timeline at divergence. Checking these files in
postgres-1, we see:

```
$ kubectl exec postgres-1 -- grep -r '' /data/batman/pg_xlog/*.history
/data/batman/pg_xlog/00000002.history:1 0/501E1D0       no recovery target specified
/data/batman/pg_xlog/00000003.history:1 0/501E698       no recovery target specified
```

This specifies that we forked from timeline 2 onto timeline 3 at 0/501E698,
which is strange because we were meant to be stuck at 0/501E628 on timeline 1.
We also jumped straight from timeline 1 to 3, via a quick stop in timeline 2.

Using `pg_xlogdump` suggests nothing different happened between postgres-0 and
postgres-1 during timeline 1. Trying to examine timeline 2 fails entirely
because we don't have any wal segments for timeline 2:

```
postgres@postgres-1:/data/batman/pg_xlog$ ls -l /data/batman/pg_xlog
total 81936
-rw------- 1 postgres postgres 16777216 Nov 21 19:46 000000010000000000000002
-rw------- 1 postgres postgres 16777216 Nov 21 19:46 000000010000000000000003
-rw------- 1 postgres postgres 16777216 Nov 21 19:46 000000010000000000000004
-rw------- 1 postgres postgres 16777216 Nov 21 20:14 000000010000000000000005
-rw------- 1 postgres postgres       41 Nov 21 20:07 00000002.history
-rw------- 1 postgres postgres       41 Nov 21 20:14 00000003.history
-rw------- 1 postgres postgres 16777216 Nov 21 20:40 000000030000000000000005
drwx------ 2 postgres postgres     4096 Nov 21 20:07 archive_status
```

I want to find out what happened and why Postgres decided to move onto timeline
3 instead of staying in timeline 2. I'm also confused by why the switch seems to
happened from an LSN that the standby Postgres didn't have available to it.
