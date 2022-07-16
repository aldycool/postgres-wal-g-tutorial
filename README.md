# PostgreSQL 14.4 with wal-g Backup and Recovery (Proof of Concept)

## Reading Materials
- https://supabase.com/blog/2020/08/02/continuous-postgresql-backup-walg
- https://wal-g.readthedocs.io/

## Steps
- DISCLAIMER: I'm using Ubuntu 20.04 (in a VM) with Docker installed for testing this. Any security measures are not considered in this tutorial (using the `/root` folder, etc.). This is just to show that the concept works, that's all. Figure out any required security yourself. Also this is all using Docker container just to not mess things up in my VM setup (no need to install PostgreSQL directly, just remove the container and images after this is done).
- To test this, we require the standard `postgres` user to be able to execute `wal-g` app, so we create a custom image of PostgreSQL 14.4 embedded with `wal-g` app inside it (the `wal-g` executable will be copied to `/usr/bin`). The `Dockerfile` file in the root folder creates this custom image, and already pushed Docker Hub as: `aldycool/postgres-wal-g:14.4`. To build:
  - Inside the `Dockerfile`, there's `wal-g` app that needs to be present before building:
    - Download `wal-g-pg-ubuntu-20.04-amd64.tar.gz` from: https://github.com/wal-g/wal-g/releases/download/v2.0.0/wal-g-pg-ubuntu-20.04-amd64.tar.gz. Extract and rename to `wal-g`:
    ```
    wget https://github.com/wal-g/wal-g/releases/download/v2.0.0/wal-g-pg-ubuntu-20.04-amd64.tar.gz
    tar -zxvf wal-g-pg-ubuntu-20.04-amd64.tar.gz
    mv wal-g-pg-ubuntu-20.04-amd64 wal-g
    rm wal-g-pg-ubuntu-20.04-amd64.tar.gz
    ```
  - Build the Docker image, and push to Docker Hub:
  ```
  docker build -t aldycool/postgres-wal-g:14.4 .
  docker push aldycool/postgres-wal-g:14.4
  ```
- The `docker-compose.yml` consists of the custom `postgres-wal-g` image and  `pgadmin4` (just in case we need to test inserting data conveniently).
- Some explanation regarding the `docker-compose.yml` values:
  - We are mounting local folder for persistence: `/root/mount/postgres` as `/var/lib/postgresql/data` inside the container, while the PostgreSQL data directory itself will be one folder on top of that: `PGDATA=/var/lib/postgresql/data/pgdata`. This is because the base folder `/var/lib/postgresql/data` will also host the wal-g test folder: `wal-g-data`. This test folder will act as the wal-g storage data. In real practice, this should be substituted with S3 or any secure supported storage systems. Read this for more detail: https://wal-g.readthedocs.io/STORAGES/.
  - `POSTGRES_DB=test_db`, we create a test database called `test_db` for inserts later.
  - `POSTGRES_PASSWORD=secretpass`, this is the minimum requirement to launch PostgreSQL container.
  - `WALG_FILE_PREFIX=/var/lib/postgresql/data/wal-g-data`, this is the target folder for wal-g to save its files.
  - The `pgadmin` section is for us to launch browser url `http://localhost:15432` (or any hostname that you setup) and login using the credentials: `admin@localhost.com` with the password `secretpass`, and then add the postgres server with url: `root_postgres_1` (the hostname assigned by docker compose network) with user `postgres` and password `secretpass`.
- Prepare some folders to keep the volume persistence:
  ```
  mkdir -p /root/mount/postgres
  mkdir -p /root/mount/pgadmin
  # pgadmin requires write access, so (lazily):
  chmod 777 /root/mount/pgadmin
  mkdir -p /root/mount/postgres/wal-g-data
  # give full write access for the wal-g test folder, so (lazily):
  chmod 777 /root/mount/postgres/wal-g-data
  ```
- Copy the `docker-compose.yml` into `/root` folder and launch the docker-compose:
  ```
  cd /root
  docker-compose up -d
  # Inspect the logs, make sure there are no errors
  docker logs root_postgres_1
  docker logs root_pgadmin_1
  ```
- Enable the PostgreSQL WAL archiving feature, edit the postgresql.conf:
  ```
  nano /root/mount/postgres/pgdata/postgresql.conf
  # Add these lines:
  archive_mode = on
  archive_command = '/usr/bin/wal-g wal-push %p'
  archive_timeout = 60
  # Restart the containers
  docker-compose down
  docker-compose up -d
  ```
- Now, we want `wal-g` to perform initial base backup. We run docker exec to invoke wal-g with `backup-push` argument as user: `postgres` 
  ```
  docker exec -it -u postgres root_postgres_1 wal-g backup-push
  ```
  This will populate the `wal-g-data` folder we setup earlier with physical backup of the current PostgreSQL directory. Take note of the time that you execute this, as we will use this time information later when restoring (my current local time is: 16 July 2022 23:14:00). Now give a few minutes before inserting into the test database.
- Create a test table in the `test_db` databasr and insert some test data (either using pgadmin or pqsql):
  ```
  docker exec -it -u postgres root_postgres_1 psql -d test_db -c "create table test(data character varying);"
  docker exec -it -u postgres root_postgres_1 psql -d test_db -c "insert into test values ('abcde');"
  ```
- NOTE: At this point, any changes that we make into databases will be pushed right away to the wal-g storage. Also, now we can create a cron job to do the `wal-g backup-push` periodically (ex. daily).
- Now we try to simulate a disaster. This is kinda tricky because we are using Docker. We will shutdown the server, but after that we call `wal-g` using docker run to perform fetch tasks, and also we need to do some modification on postgresql.conf while the server itself is down. Luckily we already persist the PostgreSQL data directory, so we will edit the file in the host machine directly.
  ```
  docker-compose down
  # Now we simulate disaster, we will delete all PostgreSQL data directory completely
  rm -rf /root/mount/postgres/pgdata/*
  # Then, we will call wal-g using docker run, to fetch base backup from wal-g storage (the parameters is exactly the same as in docker-compose.yml). The result is we will populate the PostgreSQL data directory with the latest backup from wal-g.
  docker run -v "/root/mount/postgres:/var/lib/postgresql/data" -e PGDATA=/var/lib/postgresql/data/pgdata -e POSTGRES_DB=test_db -e POSTGRES_PASSWORD=secretpass -e WALG_FILE_PREFIX=/var/lib/postgresql/data/wal-g-data --rm aldycool/postgres-wal-g:14.4 wal-g backup-fetch /var/lib/postgresql/data/pgdata LATEST
  # After the data directory gets populated, edit the postgresql.conf:
  nano /root/mount/postgres/pgdata/postgresql.conf
  # Add this line:
  restore_command = '/usr/bin/wal-g wal-fetch "%f" "%p" >> /tmp/wal.log 2>&1'
  # And also create a special file using postgres user to instruct PostgreSQL to recover using the above settings upon startup
  docker run -v "/root/mount/postgres:/var/lib/postgresql/data" -e PGDATA=/var/lib/postgresql/data/pgdata -e POSTGRES_DB=test_db -e POSTGRES_PASSWORD=secretpass -e WALG_FILE_PREFIX=/var/lib/postgresql/data/wal-g-data --rm --user postgres aldycool/postgres-wal-g:14.4 touch /var/lib/postgresql/data/pgdata/recovery.signal
  # Start the PostgreSQL again
  docker-compose up -d
  # Now check the database, any changes will be restored because we use restore_command earlier as the point of recovery.
  ```
- If, we want to restore to a specific point in time, let say the time information that we've saved earlier where we haven't create a table and add data, redo the previous steps (simulate disaster), but, the `restore_command` line should be added with these lines:
  ```
  # The saved time information earlier should be converted to UTC first
  recovery_target_time = '2022-07-16 16:15:00.000000+00'
  recovery_target_action = 'promote'
  ```
- The resulted database should be empty (as this is the time when we've just restarted to enable WAL archiving).







