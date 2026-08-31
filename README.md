# PostgreSQL 16 → PostgreSQL 18: Side-by-Side vs Same-Host In-Place Upgrade

## Overview

When planning a PostgreSQL **16 → 18 major-version upgrade on Ubuntu 26.04**, the terms **Side-by-Side Upgrade** and **In-Place Upgrade** can create confusion.

The most important point is:

> PostgreSQL does **not** support a major-version upgrade by simply replacing PostgreSQL 16 binaries with PostgreSQL 18 binaries and continuing to use the PostgreSQL 16 data files.

A PostgreSQL major-version upgrade requires a supported migration mechanism such as:

* `pg_upgrade`
* `pg_dump` / `pg_restore`
* Logical replication

For a fast same-server major upgrade, `pg_upgrade` is normally the relevant mechanism.

---

# 1. What Does "In-Place" Mean Here?

For this document, **In-Place Upgrade** means:

> PostgreSQL 16 is upgraded to PostgreSQL 18 on the **same Ubuntu 26.04 server**.

It does **not** mean:

```text
PostgreSQL 16
     |
     | apt upgrade
     v
PostgreSQL 18

Same old database files used directly
```

That is not a supported PostgreSQL major-version upgrade procedure.

Instead, PostgreSQL 18 binaries must be installed and PostgreSQL's major-upgrade mechanism must migrate the cluster.

Conceptually:

```text
Same Ubuntu 26.04 Server

PostgreSQL 16
     |
     | pg_upgrade
     v
PostgreSQL 18
```

The application can ultimately continue connecting to the same server and production port.

---

# 2. Why Are Side-by-Side and In-Place Steps Similar?

The procedures share many steps because both can ultimately use PostgreSQL's:

```text
pg_upgrade
```

The underlying database compatibility requirements therefore remain largely the same.

Both approaches require validation of:

```text
PostgreSQL health
Databases
Roles
Tablespaces
Extensions
Shared libraries
Data checksums
Locale
Collations
Replication
WAL archiving
Backups
Configuration
Statistics
Indexes
Application compatibility
Monitoring
HA/DR
Rollback
```

The major difference is primarily the **deployment topology and operational cutover**, not the PostgreSQL compatibility rules.

---

# 3. Side-by-Side Upgrade

A side-by-side architecture can maintain PostgreSQL 16 and PostgreSQL 18 as separately managed source and target environments during the migration process.

Example:

```text
Application
     |
     v

PostgreSQL 16
Source
     |
     | Major Upgrade / Migration
     v
PostgreSQL 18
Target
```

Depending on the architecture, the target could be prepared independently and the application endpoint may require a cutover.

---

# 4. Same-Host In-Place Upgrade

A same-host upgrade keeps the database service on the same Ubuntu server.

Before:

```text
Ubuntu 26.04
Server: pgdb01

Application
     |
     | TCP 5432
     v
PostgreSQL 16
```

During preparation:

```text
Ubuntu 26.04
Server: pgdb01

+--------------------------+
| PostgreSQL 16 binaries   |
+--------------------------+

+--------------------------+
| PostgreSQL 18 binaries   |
+--------------------------+

              |
              v

          pg_upgrade
```

After successful upgrade:

```text
Ubuntu 26.04
Server: pgdb01

Application
     |
     | TCP 5432
     v
PostgreSQL 18
```

The server identity can remain unchanged.

---

# 5. Comparison

| Area                             | Side-by-Side                         | Same-Host In-Place                      |
| -------------------------------- | ------------------------------------ | --------------------------------------- |
| PostgreSQL source                | PostgreSQL 16                        | PostgreSQL 16                           |
| PostgreSQL target                | PostgreSQL 18                        | PostgreSQL 18                           |
| Upgrade engine                   | `pg_upgrade` where applicable        | `pg_upgrade`                            |
| Ubuntu helper                    | `pg_upgradecluster` where applicable | `pg_upgradecluster`                     |
| PG18 binaries required           | Yes                                  | Yes                                     |
| PG16 binaries retained initially | Yes                                  | Yes                                     |
| Pre-upgrade checks               | Required                             | Required                                |
| Extension validation             | Required                             | Required                                |
| Checksum validation              | Required                             | Required                                |
| Locale/collation validation      | Required                             | Required                                |
| Backup                           | Required                             | Required                                |
| Application downtime             | Depends on migration design          | Required for final `pg_upgrade` cutover |
| Post-upgrade statistics          | Required/recommended                 | Required/recommended                    |
| Application validation           | Required                             | Required                                |
| Backup validation                | Required                             | Required                                |
| HA/DR validation                 | Required                             | Required                                |
| Rollback planning                | Required                             | Required                                |
| Same Ubuntu server               | Not necessarily                      | **Yes**                                 |
| Same server endpoint             | Not necessarily                      | Normally **Yes**                        |
| New server required              | Possible                             | **No**                                  |

---

# 6. What Remains the Same?

Approximately **60–70% of the technical validation workflow can be similar** because PostgreSQL compatibility requirements do not disappear simply because the upgrade occurs on the same server.

Common phases include:

```text
Prechecks
   |
   v
Database Inventory
   |
   v
Extension Compatibility
   |
   v
Checksum Validation
   |
   v
Locale / Collation Validation
   |
   v
Backup
   |
   v
pg_upgrade Compatibility
   |
   v
Major Upgrade
   |
   v
Statistics
   |
   v
Index / Object Validation
   |
   v
Application Validation
   |
   v
Backup Validation
   |
   v
HA / Replication Validation
   |
   v
Monitoring
   |
   v
Rollback Window
```

---

# 7. What Is Different?

The same-host procedure needs special attention to:

* PostgreSQL 16 and PostgreSQL 18 packages coexisting temporarily.
* Local filesystem capacity.
* PostgreSQL cluster ports during preparation.
* Ubuntu `postgresql-common` cluster management.
* Application endpoint preservation.
* Service startup and shutdown ordering.
* Rollback on the same host.
* PostgreSQL 16 cluster retention.
* Removal of PostgreSQL 16 only after PostgreSQL 18 acceptance.
* Removal of old PostgreSQL 16 packages as a separate cleanup activity.

---

# 8. Ubuntu 26.04 Cluster Layout

A typical Ubuntu PostgreSQL installation separates binaries, configuration, and data.

PostgreSQL 16:

```text
Binaries:
/usr/lib/postgresql/16/bin

Configuration:
/etc/postgresql/16/main

Data:
/var/lib/postgresql/16/main
```

PostgreSQL 18:

```text
Binaries:
/usr/lib/postgresql/18/bin

Configuration:
/etc/postgresql/18/main

Data:
/var/lib/postgresql/18/main
```

This is why Ubuntu/Debian PostgreSQL cluster-management tooling must be considered when designing the upgrade procedure.

---

# 9. Typical Same-Host Upgrade Flow

```text
PostgreSQL 16 Production
          |
          v
Health Checks
          |
          v
Database / Role / Tablespace Inventory
          |
          v
Extension Compatibility
          |
          v
Checksum + Locale + Collation Checks
          |
          v
Backup + Restore Validation
          |
          v
Install PostgreSQL 18
          |
          v
Upgrade Rehearsal
          |
          v
Maintenance Window
          |
          v
Stop Application Writes
          |
          v
Final Recovery Point
          |
          v
Stop PostgreSQL 16
          |
          v
pg_upgrade / pg_upgradecluster
          |
          v
PostgreSQL 18
          |
          v
Post-Upgrade Processing
          |
          v
Statistics
          |
          v
Database Validation
          |
          v
Application Smoke Testing
          |
          v
Backup + WAL Validation
          |
          v
HA / Replication Validation
          |
          v
Monitoring
          |
          v
Production Acceptance
          |
          v
Rollback Window
          |
          v
PostgreSQL 16 Decommission
```

---

# 10. Ubuntu Upgrade Command

For an Ubuntu package-managed cluster, the upgrade can use Ubuntu's cluster-management tooling with the PostgreSQL upgrade method explicitly selected.

Example:

```bash
sudo pg_upgradecluster \
  -v 18 \
  --method=upgrade \
  16 main
```

The actual production command must match the real:

* Cluster name
* PostgreSQL packages
* Data locations
* Tablespaces
* Extensions
* Filesystem
* Port configuration
* Backup strategy

Do not copy the command into production without completing the required prechecks.

---

# 11. What You Should NOT Do

Do not assume this performs a PostgreSQL major upgrade:

```bash
sudo apt update
sudo apt upgrade
```

Package upgrades alone do not convert a PostgreSQL 16 database cluster into a PostgreSQL 18 database cluster.

Also avoid:

```text
Installing PG18 and pointing it directly at PG16 PGDATA
```

or:

```text
Replacing PG16 binaries with PG18 binaries
```

PostgreSQL major releases can contain incompatible internal storage-format changes.

Use a supported major-version migration mechanism.

---

# 12. Rollback Is Important

The PostgreSQL 16 cluster should not be immediately deleted after PostgreSQL 18 starts successfully.

Recommended sequence:

```text
Upgrade PG16 → PG18
        |
        v
Technical Validation
        |
        v
Application Validation
        |
        v
Backup Validation
        |
        v
HA/DR Validation
        |
        v
Performance Monitoring
        |
        v
Acceptance
        |
        v
Rollback Window
        |
        v
Remove PG16
```

A database starting successfully does not by itself prove that the upgrade is complete.

---

# 13. Data Divergence After Cutover

Once applications begin writing to PostgreSQL 18:

```text
PostgreSQL 16 data
        !=
PostgreSQL 18 data
```

New PostgreSQL 18 transactions are not automatically written back into the PostgreSQL 16 cluster.

For this reason, establish a clear **Go/No-Go checkpoint** before opening unrestricted production writes.

---

# 14. Special Warning for Link Mode

`pg_upgrade --link` can substantially reduce upgrade time and additional disk requirements.

However, rollback becomes more complicated.

Once PostgreSQL 18 starts using linked files, the old PostgreSQL 16 cluster must not simply be assumed safe to restart.

Production use of link mode therefore requires:

```text
Verified backup/snapshot
        +
Tested rollback
        +
Documented Go/No-Go point
```

Do not select link mode solely because it is faster.

---

# 15. Recommended Terminology

For documentation, change tickets, GitHub repositories, and production SOPs, a clearer title is:

> **PostgreSQL 16 → PostgreSQL 18 Same-Host Major Version Upgrade on Ubuntu 26.04**

If the term **In-Place Upgrade** is used, define it explicitly:

> In-place in this documentation means upgrading PostgreSQL on the **same Ubuntu host**. It does not mean directly overwriting PostgreSQL 16 binaries or using PostgreSQL 16 data files with PostgreSQL 18. PostgreSQL 18 binaries are installed and PostgreSQL's supported major-version upgrade mechanism is used to convert the cluster.

---

# 16. Recommended Production Strategy

For a normal Ubuntu 26.04 same-host environment:

```text
PG16
 |
 +--> Validate
 |
 +--> Backup
 |
 +--> Install PG18
 |
 +--> Rehearse
 |
 +--> Stop Application
 |
 +--> Final Recovery Point
 |
 +--> Stop PG16
 |
 +--> Major Upgrade
 |
 +--> Start PG18
 |
 +--> Post-Upgrade Processing
 |
 +--> Validate
 |
 +--> Open Application
 |
 +--> Monitor
 |
 +--> Take PG18 Backup
 |
 +--> Retain PG16
 |
 +--> Close Rollback Window
 |
 +--> Remove PG16
```

---

# 17. Final Takeaway

The **Side-by-Side** and **Same-Host In-Place** procedures are not completely separate PostgreSQL upgrade technologies.

Both can use PostgreSQL's `pg_upgrade` mechanism.

Therefore, many technical steps naturally remain the same.

The major differences are:

```text
Deployment topology
Server placement
Storage planning
Port management
Application endpoint
Cutover procedure
Rollback design
Old-cluster cleanup
```

For PostgreSQL on Ubuntu, never interpret **in-place major upgrade** as:

```text
apt upgrade PostgreSQL 16 → PostgreSQL 18
```

Instead, think of it as:

```text
Same Ubuntu Host
       +
PG16 and PG18 binaries temporarily coexist
       +
Supported PostgreSQL major upgrade
       +
Same production endpoint after cutover
```

---

# Official References

## PostgreSQL

PostgreSQL 18 — Upgrading a PostgreSQL Cluster
[https://www.postgresql.org/docs/18/upgrading.html](https://www.postgresql.org/docs/18/upgrading.html)

PostgreSQL 18 — `pg_upgrade`
[https://www.postgresql.org/docs/18/pgupgrade.html](https://www.postgresql.org/docs/18/pgupgrade.html)

PostgreSQL 18 — Release Notes
[https://www.postgresql.org/docs/18/release-18.html](https://www.postgresql.org/docs/18/release-18.html)

PostgreSQL 18 — Data Checksums
[https://www.postgresql.org/docs/18/checksums.html](https://www.postgresql.org/docs/18/checksums.html)

PostgreSQL 18 — `initdb`
[https://www.postgresql.org/docs/18/app-initdb.html](https://www.postgresql.org/docs/18/app-initdb.html)

## Ubuntu Local Documentation

On the Ubuntu server, validate the installed `postgresql-common` behavior using:

```bash
man pg_upgradecluster
man pg_ctlcluster
man pg_lsclusters
man pg_dropcluster
```

---

## Important

Always test a PostgreSQL major-version upgrade against a production-like copy before executing it against the production database.

Extension compatibility, filesystem behavior, database size, tablespaces, replication architecture, backup software, application behavior, and available maintenance time can materially affect the final upgrade procedure.

This is ready to save directly as **`README.md`** in your GitHub repository.
