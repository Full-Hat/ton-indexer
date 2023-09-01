# TON Indexer

TON Indexer stores blocks, transactions, messages, NFTs, Jettons and DNS domains in PostgreSQL database and provides convenient API.

TON node stores data in a key-value database RocksDB. This database support only simple queries and doesn't support any kind of data filtering. An SQL database perfectly suitable for storage and retrieval data. We insert data with SQL transactions and using masterchain blocks as an atomic unit of insert to guarantee a data integrity. Indexes allows to quickly filter required data in a database.

TON Indexer stack consists of:
1. `postgres`: PostgreSQL server to store indexed data and perform queries.
2. `index-api`: FastAPI server with convinient endpoints to access database.
3. `alembic`: alembic service to run database migrations.
4. `index-worker`: TON Index worker to read and parse data from TON node database.

## How to run

Requirements:
* Docker and Docker compose (see [instruction](https://docs.docker.com/engine/install/)).
* Running TON full node (archival for full indexing).
* Recommended hardware: 
  * Database and API: 4 CPU, 32 GB RAM, 200GB disk, SSD recommended (more than 1TB required for archival indexing).
  * Worker: 4 CPU, 32 GB RAM, SSD recommended (for archival: 8 CPUs, 64 GB RAM, SSD recommended).
* To increase history indexing performance: 
  * Remove PostgreSQL indexes: `docker compose run --rm alembic alembic downgrade -1`.
  * Create indexes after indexer will catch up TON network: `docker compose up alembic`.

Do the following steps to setup TON Indexer:
* Clone repository: `git clone --recursive --branch cpp-indexer https://github.com/kdimentionaltree/ton-indexer`.
  * If you've forgot `--recursive` flag you can use command: `git submodule update --recursive --init`.
* Create *.env* file with command `./configure.sh`.
  * Run `./configure.sh --worker` to configure TON Index worker.
* Adjust parameters in *.env* file (see [list of available parameters](#available-parameters)).
* Build docker images: `docker compose build postgres alembic index-api`.
* Run stack: `docker compose up -d postgres alembic index-api`.
  * To start worker use command `docker compose up -d worker` after creating all services.

**NOTE:** we recommend use to setup indexer stack and index worker on a separate servers. To install index worker to **Systemd** check this [instruction](https://github.com/kdimentionaltree/ton-index-cpp).

### Available parameters

* PostgreSQL parameters:
  * `POSTGRES_HOST`: PostgreSQL host. Can be IP or FQDN. Default: `postgres`.
  * `POSTGRES_PORT`: PostgreSQL port. Default: `5432`.
  * `POSTGRES_USER`: PostgreSQL port. Default: `postgres`.
  * Set password in file `private/postgres_password`.
  * `POSTGRES_DBNAME`: PostgreSQL database name. You can change database name to use one instance of PostgreSQL for different TON networks (but we strongly recommend to use separate instances). Default: `ton_index`.
  * `POSTGRES_PUBLISH_PORT`: a port to publish. Change this port if you host multiple indexers on the same host. Default: `5432`.
* API parameters:
  * `TON_INDEXER_API_ROOT_PATH`: root path for reverse proxy setups. Keep it blank if you don't use reverse proxies. Default: `<blank>`.
  * `TON_INDEXER_API_PORT`: a port to expose. You need check if this port is busy. Use different ports in case of multiple indexer instances on the same host. Default: `8081`.
  * `TON_INDEXER_WORKERS`: number of API workers. Default: `4`.
* TON worker parameters:
  * `TON_WORKER_DBROOT`: path to TON full node database. Use default value if you've installed node with `mytonctrl`. Default: `/var/ton-work/db`.
  * `TON_WORKER_FROM`: masterchain seqno to start indexing. Set 1 to full index, set last masterchain seqno to collect only new blocks (use [/api/v2/getMasterchainInfo](https://toncenter.com/api/v2/getMasterchainInfo) to get last masterchain block seqno).
  * `TON_WORKER_MAX_PARALLEL_TASKS`: max parallel reading actors. Adjust this parameter to decrease RAM usage. Default: `1024`.
  * `TON_WORKER_INSERT_BATCH_SIZE`: max masterchain seqnos per INSERT query. Small value will decrease indexing performance. Great value will increase RAM usage. Default: `512`.
  * `TON_WORKER_INSERT_PARALLEL_ACTORS`: number of parallel INSERT transactions. Increasing this number will increase PostgreSQL server RAM usage. Default: `3`.

# FAQ

## How to point an TON Index worker to existing PostgreSQL instance
* Remove PostgreSQL container: `docker compose rm postgres` (add flag `-v` to remove volumes).
* Setup PostgreSQL credentials in *.env* file.
* Run alembic migration: `docker compose up alembic`.
* Run index worker: `docker compose index-worker`.

## How to update code
* Pull new commits: `git pull`.
* Update submodules: `git submodule update --recursive --init`.
