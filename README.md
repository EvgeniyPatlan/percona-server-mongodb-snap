# percona-server-mongodb

Percona Server for MongoDB packaged as a strict-confinement snap: the
server, the `mongos` sharding router, MongoDB Shell (`mongosh`) plus the
legacy `mongo` shell, the MongoDB database tools, and Percona Backup for
MongoDB (PBM) — staged unmodified from Percona's official apt packages
(`repo.percona.com/psmdb-80` or `psmdb-83`, plus the separate `pbm`
repository). Two MongoDB majors are published as separate branches/tracks,
each pinned to an exact upstream package version (see below). Base:
`core24`.

## Why this snap

Installing this snap gets the whole Percona Server for MongoDB stack —
server, sharding router, shells, tools, and backup agent — in one
artifact, with every component pinned to an exact upstream version.
`mongod` starts automatically as a standalone server on `127.0.0.1:27017`;
`mongos` and `pbm-agent` are installed but left disabled, because both
need cluster or replica-set context that install time can't supply on its
own. Each supported MongoDB major (8.0 LTS, 8.3 rapid) is a distinct
branch/track, so tracking a rapid release is an explicit channel choice
rather than an automatic upgrade.

## Tracks and branches

| Branch | apt source | Version |
|---|---|---|
| `8.0/edge` | `repo.percona.com/psmdb-80/apt` (noble, main) | 8.0.29-13 |
| `8.3/edge` | `repo.percona.com/psmdb-83/apt` (noble, main) | 8.3.7-1 |

8.3 is a MongoDB rapid release: when Percona ships the next rapid
(`psmdb-84`, …), this branch's repo URL, version, and package pins move
together — `psmdb-82` has already been retired upstream. Both tracks also
stage `percona-backup-mongodb` from the separate
`repo.percona.com/pbm/apt` repository.

## Getting the snap

### From a CI build

Every push to a `*/edge` branch, every pull request, and every manual
`workflow_dispatch` run of the `Tests` workflow builds the snap (amd64 and
arm64) and runs the full spread suite against it.

1. Open the workflow run in GitHub Actions and download the
   `snap-packages` artifact.
2. Unzip it.
3. Install:
   ```
   sudo snap install ./percona-server-mongodb_<version>_amd64.snap --dangerous --jailmode
   ```
   (substitute the `arm64` filename on that architecture).

### From source

```
git clone https://github.com/EvgeniyPatlan/percona-server-mongodb-snap.git
cd percona-server-mongodb-snap
git checkout 8.0/edge   # or 8.3/edge
snapcraft pack
sudo snap install ./percona-server-mongodb_*.snap --dangerous --jailmode
```

Requires the `snapcraft` and `lxd` snaps.

Store channels exist for each track (`8.0/edge`, `8.3/edge`), but the
release workflow only publishes when the repository's `RELEASE_ENABLED`
variable is set, so Store availability isn't guaranteed.

## First steps

```
percona-server-mongodb.mongosh
```

Security posture (as installed): the standalone server starts with
authorization **disabled** — any local process can read and write via
`127.0.0.1:27017`. Enable the security section in `mongod.conf` (see
below) before putting data you care about on it.

## Services and apps

| App | Kind | Purpose |
|---|---|---|
| `mongod` | daemon, auto-started | MongoDB server, standalone on `127.0.0.1:27017` |
| `mongos` | daemon, disabled by default | sharding query router, `127.0.0.1:27018` |
| `pbm-agent` | daemon, disabled by default | Percona Backup for MongoDB agent |
| `mongosh` | CLI | MongoDB Shell |
| `mongo` | CLI | legacy MongoDB shell |
| `pbm` | CLI | Percona Backup for MongoDB control tool |
| `mongodump` | CLI | logical backup |
| `mongorestore` | CLI | logical restore |
| `mongoexport` | CLI | export a collection to JSON/CSV |
| `mongoimport` | CLI | import JSON/CSV into a collection |
| `mongostat` | CLI | live server activity stats |
| `mongotop` | CLI | live per-collection read/write activity |
| `mongofiles` | CLI | manipulate GridFS-stored files |
| `bsondump` | CLI | convert a BSON dump file to JSON |
| `get-keyfile` | CLI | print the replica-set keyfile, to copy it to other members |
| `set-keyfile` | CLI | install a provided replica-set keyfile |

Starting the optional daemons:

```
sudo snap start percona-server-mongodb.mongos
sudo snap start percona-server-mongodb.pbm-agent
sudo snap stop percona-server-mongodb.mongod
sudo snap restart percona-server-mongodb.mongod
```

`snap set` configuration knobs:

| Key | Effect |
|---|---|
| `mongod-args` | extra CLI flags appended to `mongod` on start |
| `mongos-args` | extra CLI flags appended to `mongos` on start |
| `pbm-uri` | `PBM_MONGODB_URI` used by `pbm-agent` and the `pbm` CLI |

## Configuration and data paths

| Item | Path |
|---|---|
| `mongod` config | `/var/snap/percona-server-mongodb/current/etc/mongod/mongod.conf` |
| `mongos` config | `/var/snap/percona-server-mongodb/current/etc/mongos/mongos.conf` |
| Data directory | `/var/snap/percona-server-mongodb/common/var/lib/mongodb` (survives snap refreshes) |
| `mongod` log | `/var/snap/percona-server-mongodb/common/var/log/mongodb/mongod.log` |
| `mongos` log | `/var/snap/percona-server-mongodb/common/var/log/mongos/mongos.log` |
| Replica-set keyfile | `/var/snap/percona-server-mongodb/current/etc/mongodb-keyfile` (generated automatically at install time) |
| Run/socket directory | `/var/snap/percona-server-mongodb/common/run` |

Input files for the CLI tools must live somewhere under
`/var/snap/percona-server-mongodb/common` — strict confinement means the
snap cannot read your home directory.

## Replica set and PBM quickstart

Enable security and replication, then initiate a single-member replica
set:

```
sudo sed -i \
  -e 's/^#security:/security:/' -e 's/^#  keyFile:/  keyFile:/' -e 's/^#  authorization:/  authorization:/' \
  -e 's/^#replication:/replication:/' -e 's/^#  replSetName:/  replSetName:/' \
  /var/snap/percona-server-mongodb/current/etc/mongod/mongod.conf

sudo snap restart percona-server-mongodb.mongod
percona-server-mongodb.mongosh --eval 'rs.initiate()'
```

Once a replica set is up, configure and run PBM:

```
sudo snap set percona-server-mongodb pbm-uri="mongodb://admin:<password>@127.0.0.1:27017/?replicaSet=rs0&authSource=admin"
percona-server-mongodb.pbm config --file /path/to/pbm-config.yaml
sudo snap start percona-server-mongodb.pbm-agent
percona-server-mongodb.pbm backup --wait
```

## Testing

Every push and pull request runs the full spread suite against a real
snapd install inside an LXD `ubuntu-24.04` VM, on both `amd64` and
`arm64`. Suites: `backup_restore`, `cli_tools`, `mongos_router`,
`replica_set`, `smoke`, and `upgrade` (currently marked `manual` until the
snap is published to `8.0/edge`).

To reproduce locally:

```
snapcraft pack
CRAFT_ARTIFACT=$(pwd)/percona-server-mongodb_<version>_amd64.snap spread -v
```

(`spread` from `go install github.com/canonical/spread/cmd/spread@latest`;
needs the `lxd` snap.)

## License

The snap packaging is Apache-2.0. Upstream component licenses are shipped
under `licenses/` inside the snap. Percona Server for MongoDB itself is
distributed under the Server Side Public License (SSPL); its disclosure
text is included as `licenses/LICENSE-percona-server-mongodb`.
