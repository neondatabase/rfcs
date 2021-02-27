# Command line interface (end-user)

Zenith CLI as it is described here mostly resides on the same conceptual level as pg_ctl/initdb/pg_recvxlog/etc and replaces some of them in an opinionated way. I would also suggest bundling our patched postgres inside zenith distribution at least at the start.

The most important concept here is a snapshot, which can be created/moved/sent/exported. Also, we may start temporary read only postgres instance over any local snapshot. A more complex scenarios would consist of several basic operations over snapshots.

## storage

Storage is either pagestore or s3. Users may create a database in a pagestore and create/move *snapshots* and *pitr regions* in both pagestore and s3. Storage is a concept similar to `git remote`. After installation, I imagine two storages are available by default: local pagestore and zenith cloud pagestore.

XXX: is there any better ideas on how to call pagestore? maybe just zenith? zenith-smgr?

**zenith storage attach** -t [pagestore|s3] -c key=value -n name

Attaches/initializes storage. For --type=s3, user credentials and path should be provided. For --type=pagestore we may support --path=/local/path and --url=zenith.tech/stas/mystore


**zenith storage list**

Show currently attached storages. For example:

```
> zenith storage list
NAME            USED    TYPE                OPTIONS
local           5.1G    pagestore-local
local.compr     20.4G   pagestore-local     comression=on
zcloud          60G     pagestore-remote
s3tank          80G     S3
```

**zenith storage detach**

**zenith storage show**



## pg

Pg is a term for a single postgres running on some data. I'm trying to avoid here separation of datadir management and postgres instance management -- both that concepst bundled here together.

**zenith pg create** [--no-start --snapshot --cow] -s storage-name -n pgdata

Creates (initializes) new data directory in given storage and starts postgres. I imagine that storage for this operation may be only local pagestore and data movement to remote location happens through snapshots/pitr.

--no-start: just init datadir without creating 

--snapshot snap: init from the snapshot. Snap is a name or URL (zenith.tech/stas/mystore/snap1)

--cow: initialize Copy-on-Write data directory on top of some snapshot (makes sense if it is a snapshot of currently running a database)

**zenith pg destroy**

**zenith pg start** [--replica] pgdata
Start postgres with proper extensions preloaded/installed.
    
**zenith pg stop** pg_id

**zenith pg list**

```
ID                   PGDATA        USED    STORAGE            ENDPOINT
master@my_pg         my_pg         5.1G    local              localhost:5432
replica-1@my_pg                                               localhost:5433
replica-2@my_pg                                               localhost:5434
master@my_pg2        my_pg2        3.2G    local.compr        localhost:5435
-                    my_pg3        9.2G    local.compr        -
```

**zenith pg show**

```
my_pg:
    storage: local
    space used on local: 5.1G
    space used on all storages: 15.1G
    snapshots:
        on local:
            snap1: 1G
            snap2: 1G
        on zcloud:
            snap2: 1G
        on s3tank:
            snap5: 2G
    pitr:
        on s3tank:
            pitr_one_month: 45G

```

**zenith pg start-rest/graphql** pgdata

Starts REST/GraphQL proxy on top of postgres master. Not sure we should do that, just an idea.


## snapshot

Snapshot creation is cheap -- no actual data is copied, we just start retaining old pages. Snapshot size means the amount of retained data, not all data.

**zenith snapshot create** pgdata_name@snap_name

Creates a new snapshot in the same storage where pgdata_name exists.

**zenith snapshot move** --to new_storage_name pgdata_name@snap_name

Moves (copy and delete src) snapshot to another storage either it is cloud or s3. Under the hood would use more low-level command send.

**zenith snapshot send** --to url

Produces binary stream of a given snapshot. Under the hood starts temp read-only postgres over this snapshot and sends basebackup info.

**zenith snapshot recv** [--from=url]

Without --from starts a port listening for a basebackup stream and expects data on that socket. With --from option connects to a remote postgres and pulls basebackup itself via replication protocol.

**zenith snapshot export**

Starts read-only postgres over this snapshot and exports data in some format (pg_dump, or COPY TO on some/all tables).

**zenith snapshot diff** snap1 snap2

Shows size of data changed between two snapshots. We also may provide options to diff schema / data in tables. To do that start temp read only postgreses.

**zenith snapshot destroy**

## pitr

Pitr represents wal stream and ttl policy for that stream

XXX: any suggestions on a better name?

**zenith pitr create** name
    --ttl = inf | period
    --size-limit = inf | limit
    --storage = storage_name

**zenith pitr extract-snapshot** pitr_name --lsn xxx
    Creates a snapshot out of some lsn in PITR area. The obtained snapshot may be managed with snapshot routines (move/send/export)

**zenith pitr gc** pitr_name
    Force garbage collection on some PITR area.

**zenith pitr list**

**zenith pitr destroy**


## console

**zenith console** opens browser targeted at web console with the more or less same functionality as described here.
