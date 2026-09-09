# helloclicks-mysql

Docker image for the `helloclicks-db` private service on Render: the stock
`mysql/mysql-server` image plus a `config/user.cnf` tuned for the 4 GB
instance the HelloClicks database runs on.

Forked from [render-examples/mysql](https://github.com/render-examples/mysql).
The only files that matter are `Dockerfile` (pins the MySQL version) and
`config/user.cnf` (the server settings). `render.yaml` is the template's
original blueprint and is not used; the service is managed in the dashboard.

## Why this exists

The template's config sets only the auth plugin and bind address, so every
memory setting was at MySQL's factory default:

| Setting | Factory default | Effect on a 4 GB box |
|---|---|---|
| `innodb_buffer_pool_size` | 128 MB | almost all reads went through the Linux file cache, ~2.8 GB |
| `temptable_max_ram` | 1 GB | scratch pool grows on demand and is never released |
| `performance_schema` | ON | ~150 MB that only grows until restart |
| `max_connections` | 151 | slots reserved for connections the app never opens |

Measured on production before the change: MySQL process 1.4 GB resident,
file cache 2.8 GB, instance at about 90%.

## Deploying

The `helloclicks-db` service points at this repo, branch `main`, runtime
Docker. Keep **Auto-Deploy off** so a commit here never restarts the database
by surprise; use **Manual Deploy** when the config changes. Each deploy
restarts MySQL once, for well under a minute; the web and Celery services
reconnect on their own.

The disk mount (`/var/lib/mysql`) and the `MYSQL_*` environment variables
live on the service, not in this repo, and are unaffected by deploys.

## Verifying

From the service's **Shell** tab:

```bash
mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "SELECT @@innodb_buffer_pool_size/1048576 AS pool_mb, @@temptable_max_ram/1048576 AS temptable_mb, @@performance_schema, @@max_connections;"
grep -H VmRSS /proc/1/status
grep -E "^(anon|file) " /sys/fs/cgroup/memory.stat
```

Expected: `pool_mb` 1536, `temptable_mb` 256, `performance_schema` 0,
`max_connections` 60. After a day of traffic the process RSS should sit
around 2 GB and stop climbing, and the `file` figure should be a small
fraction of what it was.

### Getting a memory breakdown later

`performance_schema` is off, so `sys.memory_global_by_current_bytes` is empty.
To investigate memory again, set `performance_schema=ON` in `config/user.cnf`,
deploy, run the query, then turn it back off.

## Upgrading MySQL (separate step)

Production runs 8.0.24 (April 2021). Newer 8.0.x releases fix several
temp-table memory bugs. Within the 8.0 series MySQL upgrades the data
directory in place on first start, but it cannot be downgraded afterwards, so
treat a version bump as its own change:

1. Confirm a recent S3 backup completed (the app's `backup_database_to_s3`
   task).
2. Change the `FROM` line in `Dockerfile` to a newer `mysql/mysql-server:8.0.x`
   tag and deploy off-peak.
3. Watch the first start in the service logs for the in-place upgrade to
   finish before traffic resumes.

Do not jump to 8.4 with this config: `temptable_use_mmap` and
`default-authentication-plugin` were removed there.
