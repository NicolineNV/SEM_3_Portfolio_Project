---
title: Entry 16 - From personal machine to the Droplet - Getteing the API deployed
date: 2026-10-28
showAuthor: true
series: ["Backend-REST-API"]
series_order: 16
---

## Moving the seeded database to the server

### The situation

The droplet already ran a shared PostgreSQL container from earlier projects. It lives in the folder `~/deployment` together with the other services: `caddy`, `portfolio` and `carport`. Each project has its own database in that one Postgres server. I reused this server instead of installing a second PostgreSQL.

The production database starts empty. My 222 seeded cocktails exist only in my local database, which also runs in Docker on my Mac (the same image as on the server, `postgres:16.2`, which avoids any dump compatibility issues).

### Why a dump

I could have run the importer against the server, but a dump is faster and gives an exact copy of the database I have already tested. It also contains the schema (created from `schema.sql`), which matters because the production configuration uses `validate`.

### The steps

1. **Dump on my Mac**, written to the Desktop so I would not commit it by mistake:
   ```bash
   docker exec db pg_dump -U postgres -d drinksportfolio --no-owner --no-privileges > ~/Desktop/cocktails.sql
   ```
   `--no-owner --no-privileges` removes ownership information. The user that owned the tables locally does not necessarily exist on the server, and the restore would otherwise fail.
2. **Copy to the server** with `scp ~/Desktop/cocktails.sql jetty@<droplet-ip>:~/`.
3. **Create the empty database** on the server:
   ```bash
   docker exec -it db psql -U postgres -c "CREATE DATABASE drinksportfolio;"
   ```
4. **Load the dump**:
   ```bash
   docker exec -i db psql -U postgres -d drinksportfolio < ~/cocktails.sql
   ```
   (`-i`, not `-it`, because a file is piped in.)
5. **Verify.** `\dt` listed all 14 tables, `COPY 222` appeared for cocktails during the load, and `SELECT count(*) FROM cocktail;` returned 222.

### Challenge: finding the right compose file

When I was told to edit the existing `docker-compose.yml`, I opened `nano docker-compose.yml` in my home folder. The editor showed an empty file with `[ New File ]` at the bottom, because the real file is in another folder. I found the real location from Docker itself, since containers started by Compose carry a label with their project folder:

```bash
docker inspect caddy --format '{{ index .Config.Labels "com.docker.compose.project.working_dir" }}'
```

This returned `/home/jetty/deployment`.

### Something to remember

The production database is not changed by Hibernate (`validate`). Any later schema change, such as new tables, must also be applied on the server manually with SQL from `schema.sql`.

---

## Entry 8: Running and exposing the API on the droplet

### Design decision: a separate compose file

Everything on the server shares the Docker network `deployment_default`, because it is the default network of the existing compose project. That means containers find each other by name (`db`, `caddy`, `portfolio`, `carport`).

I could have added my API to the existing `docker-compose.yml`, but that file also runs my two older projects. A single indentation error in it could affect them the next time it is applied. I therefore put the new API in its own folder with its own compose file and attached it to the existing network as an **external** network:

```yaml
services:
  cocktail_api:
    image: <dockerhub-user>/cocktail_api:latest
    container_name: cocktail_api
    restart: unless-stopped
    expose:
      - "7070"
    environment:
      DEPLOYED: "true"
      CONNECTION_STR: jdbc:postgresql://db:5432/
      DB_NAME: drinksportfolio
      DB_USERNAME: postgres
      DB_PASSWORD: ${POSTGRES_PASSWORD}
      SECRET_KEY: ${SECRET_KEY}
      ISSUER: ${ISSUER}
      TOKEN_EXPIRE_TIME: ${TOKEN_EXPIRE_TIME}
    labels:
      - com.centurylinklabs.watchtower.enable=true
    networks:
      - deployment_default

networks:
  deployment_default:
    external: true
```

Important points:

- **`db` as the host name.** Inside a container, `localhost` means the container itself. The database is another container, found by its name on the shared network. The `CONNECTION_STR` ends with `/` because my `HibernateConfig` appends `DB_NAME` to it.
- **`expose` instead of `ports`.** The port is reachable by other containers (Caddy) but not from the internet.
- **`${...}` references and a `.env` file.** The real values are in a `.env` file next to the compose file, with permissions limited by `chmod 600`. The `SECRET_KEY` (for the future JWT tokens) was generated with `openssl rand -hex 32`. The example key from the tutorial must not be used, since it is public.
- **A Watchtower label**.

### HTTPS and the domain

Caddy is the reverse proxy for all my sites and gets HTTPS certificates automatically. Two things were needed to add the API:

1. **DNS.** I created an A record for the subdomain `drinksapi` pointing to the droplet's IP address in DigitalOcean's DNS. Without it, Caddy cannot validate the domain and cannot issue a certificate.
2. **A new block in the Caddyfile**:
   ```
   drinksapi.nvisby.com {
       reverse_proxy cocktail_api:7070
   }
   ```
   followed by `docker exec caddy caddy reload --config /etc/caddy/Caddyfile`.

The reload printed a warning that the Caddyfile "is not formatted". This only concerns indentation and spacing. It is not an error, and the log level was `warn`, not `error`.

### Challenge: HTTP 502 with an empty response

My first `curl` command printed nothing at all. Plain `curl` does not show the status line or empty bodies. Adding `-i` revealed `HTTP/2 502` with `content-length: 0` and `server: Caddy`. That is Caddy's way of saying that it is running but cannot reach the service behind it.

The container itself was fine: its log showed `HikariPool-1 - Added connection` (so the database connection and the environment variables were correct) and `Javalin started`. The remaining suspect was Caddy's route to the container. The target in `reverse_proxy` has to be the container's exact name, and it must be on the same network. **[Fill in yourself: what you corrected in the Caddyfile, for example the container name.]**

### Challenge: the base path

The Javalin log also stated `Listening on http://localhost:7070/api/v1`. The working address is therefore `/api/v1/cocktails/1` and not `/api/cocktails/1`. The same base path has to be used in the REST-assured tests.

### Result

```
$ curl -i https://drinksapi.nvisby.com/api/v1/cocktails/1
HTTP/2 200
content-type: application/json
{"id":1,"name":"A1", ... "spirit":"Gin","liqueur":"Grand Marnier", ...}
```

The seeded data from the server database is served over HTTPS through Caddy.

### Security notes

- All secrets are in environment variables or in `.env`, which is never committed.
- The Postgres container from my earlier setup publishes port 5432 on all interfaces (`0.0.0.0:5432`), which makes the database reachable from the internet. I did not change it, since it belongs to the other projects' compose file, but the safer setting is `127.0.0.1:5432:5432`, together with a strong password.


## Automatic redeployment with Watchtower

### How it works

Watchtower is a container that asks Docker Hub at an interval (here 60 seconds) whether the images of running containers have a newer version. If they do, it downloads it and restarts the container. Together with the pipeline from Entry 6, the full loop is:

```
git push  ->  Actions: test + build + push image  ->  Watchtower pulls it  ->  new container
```

The option `--label-enable` makes Watchtower touch only containers with the label `com.centurylinklabs.watchtower.enable=true`. That is only the API. The older projects (`portfolio` and `carport`), the database and Caddy are never restarted by it.

### Challenge: Watchtower restarting in a loop

`docker ps` showed Watchtower as `Restarting (1)`. Its log repeated the same line every minute:

```
Error response from daemon: client version 1.25 is too old. Minimum supported API version is 1.40
```

The Watchtower image uses an old version of the Docker API client (1.25), and the Docker daemon on the droplet no longer accepts anything below 1.40. It is a compatibility problem between the tool and the server, not a mistake in my configuration. The fix is to tell Watchtower to use a newer API version with an environment variable:

```yaml
watchtower:
  image: containrrr/watchtower
  environment:
    DOCKER_API_VERSION: "1.40"
```

1.40 is the lowest version the server accepts, so it is guaranteed to be allowed (a higher value, such as 1.44, is the alternative if the log still complains). As far as I know, the Watchtower project is no longer actively maintained, which is a reason to expect similar trouble in the future and to look at maintained forks if it happens.
