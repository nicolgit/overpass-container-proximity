# Overpass API for Italy - Docker

A local Overpass API instance containing OpenStreetMap data for Italy.

The setup uses:

- Docker Compose
- `wiktorn/overpass-api`
- Geofabrik's OpenStreetMap extract for Italy
- A persistent local database
- Automatic incremental updates

> **Note:** Overpass Turbo is a web interface for running Overpass queries. This
> container runs **Overpass API**, the backend queried through
> `/api/interpreter`.

---

## Project structure

```text
overpass-api/
├── docker-compose.yaml
├── README.md
└── overpass-db/
```

The container creates and uses `overpass-db` to store the persistent database.

---

## Prerequisites

Install:

- Docker
- Docker Compose v2

Check the installed versions:

```bash
docker --version
docker compose version
```

---

## Start the service

From the project directory, run:

```bash
docker compose up -d
```

To follow the logs:

```bash
docker compose logs -f overpass
```

On the first startup with an empty database, the container downloads the Italian
extract:

```text
https://download.geofabrik.de/europe/italy-latest.osm.pbf
```

The PBF file is preprocessed with `osmium` and imported into the Overpass
database. The initial import can take a long time, depending particularly on
CPU, memory, and disk performance.

---

## API endpoint

After initialization is complete, the Overpass API is available at:

```text
http://localhost:8080/api/interpreter
```

---

## Test the API

Run a query for nodes tagged as cities and named Rome:

```bash
curl.exe "http://localhost:8080/api/interpreter" --data-urlencode 'data=[out:json];node["place"="city"]["name"="Roma"];out;'
```

The response should contain matching OpenStreetMap data for Rome, if present in
the imported extract.

---

## OpenStreetMap updates

The database is not limited to the initial snapshot. Incremental updates are
configured to come from Geofabrik's Italy updates feed:

```text
https://download.geofabrik.de/europe/italy-updates/
```

The setting:

```yaml
OVERPASS_UPDATE_SLEEP: "3600"
```

sets the delay between update attempts to 3,600 seconds (one hour). This is the
polling interval, not a guarantee that new data will be available every hour;
the actual update frequency depends on when Geofabrik publishes new diffs.
Available diffs are applied incrementally to the existing database, so the full
Italian extract does not need to be downloaded and imported again for each
update.

---

## Persistence

The Overpass database is persisted with this bind mount:

```yaml
volumes:
  - ./overpass-db:/db
```

This separates the container lifecycle from the database:

```text
Docker image
     |
Overpass container       can be recreated or updated
     |
./overpass-db            remains persistent
```

You can stop or recreate the container without deleting the database. For
example:

```bash
docker compose down
docker compose up -d
```

The existing database in `overpass-db` is retained, so a full reimport should
not be necessary.

### Important

Do not delete `./overpass-db` if you want to keep the database and update state.

---

## Update the Docker image

You do not need to manually remove the old image. Run:

```bash
docker compose pull
docker compose up -d
```

`docker compose pull` downloads the image referenced by the configured image
tag. `docker compose up -d` recreates the container when needed, using that
image while keeping the database in the persistent directory.

Alternatively:

```bash
docker compose up -d --pull always
```

This tells Docker Compose to check for the image referenced by the configured
tag before starting the service.

---

## Remove unused Docker images

After an image update, old images may remain on the system. By default,
`docker image prune` removes only dangling images; it does not remove every
unused, tagged image.

```bash
docker image prune
```

Images referenced by containers are not removed.

---

## Stop the service

Stop and remove the container:

```bash
docker compose down
```

The database remains in:

```text
./overpass-db
```

Start the service again with:

```bash
docker compose up -d
```

---

## Check container status

```bash
docker compose ps
```

To view the logs:

```bash
docker compose logs overpass
```

To follow them in real time:

```bash
docker compose logs -f overpass
```

---

## Restart the service

```bash
docker compose restart overpass
```

---

## Main configuration

The `docker-compose.yaml` file uses the following settings:

```yaml
environment:
  OVERPASS_MODE: init

  OVERPASS_PLANET_URL: "https://download.geofabrik.de/europe/italy-latest.osm.pbf"

  OVERPASS_PLANET_PREPROCESS: "osmium cat -f osm.bz2"

  OVERPASS_DIFF_URL: "https://download.geofabrik.de/europe/italy-updates/"

  OVERPASS_UPDATE_SLEEP: "3600"

  OVERPASS_META: "no"
  OVERPASS_COMPRESSION: "lz4"

  OVERPASS_RULES_LOAD: "10"

  OVERPASS_STOP_AFTER_INIT: "false"
```

### Key parameters

| Variable | Purpose |
|---|---|
| `OVERPASS_MODE` | Selects initialization mode; `init` imports the configured OSM extract. |
| `OVERPASS_PLANET_URL` | URL of the initial OSM extract for Italy. |
| `OVERPASS_PLANET_PREPROCESS` | Command used to preprocess the PBF file before import. |
| `OVERPASS_DIFF_URL` | Source directory for incremental updates. |
| `OVERPASS_UPDATE_SLEEP` | Delay, in seconds, between update attempts. |
| `OVERPASS_META` | Controls whether OSM metadata is retained; `no` disables it. |
| `OVERPASS_COMPRESSION` | Database compression mode. |
| `OVERPASS_RULES_LOAD` | Controls the work/sleep balance during area generation; higher values allocate more time to processing. |
| `OVERPASS_STOP_AFTER_INIT` | When `false`, keeps the container running after initialization. |

---

## Recommended periodic image update

To periodically update the Docker image:

```bash
cd overpass-api

docker compose pull
docker compose up -d
docker image prune
```

The database is not removed by these commands because it is stored in
`./overpass-db`. The prune command removes dangling images only, unless
additional options are specified.

---

## Backup

To back up the instance, preserve at least:

```text
docker-compose.yaml
overpass-db/
```

For a consistent backup, stop the container first:

```bash
docker compose down
tar -czf overpass-backup.tar.gz docker-compose.yaml overpass-db/
docker compose up -d
```

---

## Notes

- The initial import of Italy can be CPU-, memory-, and I/O-intensive.
- An SSD or NVMe drive is strongly recommended.
- Complex Overpass queries can require significant amounts of memory.
- The local database can use considerably more disk space than the original
  `.osm.pbf` file.
- Do not delete `overpass-db` unless you intend to rebuild the database from
  scratch.

## References

- [wiktorn/overpass-api image documentation](https://github.com/wiktorn/overpass-api)
- [Geofabrik Italy downloads](https://download.geofabrik.de/europe/italy.html)
- [Overpass API User's Manual](https://dev.overpass-api.de/overpass-doc/en/)
