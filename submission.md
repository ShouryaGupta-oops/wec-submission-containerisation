For phase 1 that is to list out the error and correct it
I first skimmed through the dockerfile and every otherfile
then i run the whole file with docker compose up --build
and i got following error message

PrismaClientInitializationError: Unable to require(`/app/node_modules/.prisma/client/libquery_engine-linux-musl.so.node`).
orbis-backend   | The Prisma engines do not seem to be compatible with your system. Please refer to the documentation about Prisma's system requirements: https://pris.ly/d/system-requirements
orbis-backend   |
orbis-backend   | Details: Error loading shared library libssl.so.1.1: No such file or directory (needed by /app/node_modules/.prisma/client/libquery_engine-linux-musl.so.node)
orbis-backend   |     at Object.loadLibrary (/app/node_modules/@prisma/client/runtime/library.js:111:9152)
orbis-backend   |     at async kr.loadEngine (/app/node_modules/@prisma/client/runtime/library.js:112:448)
orbis-backend   |     at async kr.instantiateLibrary (/app/node_modules/@prisma/client/runtime/library.js:111:11508)
orbis-backend   |     at async kr.start (/app/node_modules/@prisma/client/runtime/library.js:112:1976)
orbis-backend   |     at async connectWithRetry (file:///app/src/config/database.js:20:7) {
orbis-backend   |   clientVersion: '6.0.1',
orbis-backend   |   errorCode: undefined
orbis-backend   |

So from 6-7 lines i understood Prisma(open source database toolkit) is failed to detect newer version of openssl library
prisma tried to load its engine and failed
alpine ships very littile by default
engine wants openSSl 1.1
the backend crashes and Docker compose restarts it, so i am seeing same error again and again

error1-Prisma requires platform-specific engine binaries.the default native target gnerates binaries for build machine not for linux-musl

so i added a command "RUN apk add --no-cache openssl" in dockerfile of backend which basically does executes a command while building image and added openssl library

and i explicitly set binary targets to["native","linus-musl-openssl-3.0.x"] as Alpine Linux uses musl libc instead of general glibc

even after correcting this error app was listening correctly on the port but i am not to see any thing on the webpage

Why this error occurs: Vite's environment variable system (import.meta.env) performs static replacement at build time, not runtime. This is fundamentally different from server-side env vars. The variables must exist when npm run build executes inside the Dockerfile.tho it is not containerization issue still i took help of ai and fixed as i am in my 2nd year only i dont a lot of stuff

i fixed it by adding
frontend:
  build:
    context: ./frontend
    args:
      VITE_AUTH0_DOMAIN: "your-domain.auth0.com"
      VITE_AUTH0_CLIENT_ID: "your-client-id"
in docker compose file

to reflect this change i added 
ARG VITE_AUTH0_DOMAIN
ARG VITE_AUTH0_CLIENT_ID
ENV VITE_AUTH0_DOMAIN=$VITE_AUTH0_DOMAIN
ENV VITE_AUTH0_CLIENT_ID=$VITE_AUTH0_CLIENT_ID
in frontend docker file

but here still there was an issue in that browser is seeing blank page which i have fixed later

When Docker sends SIGTERM to stop the container (during docker-compose down), the graceful shutdown crashes, and the container is force-killed after the timeout (default 10s).
app.listen() returns an http.Server instance, but it's not assigned to any variable.The gracefulShutdown function references server which doesn't exist in scope.

so to fix it
app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
changes to this:

const server = app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});

so now i head towards to phase 2
i first build the image of frontend dockerfile
and find that it is using 1.9 gb on the disk so i have to lower it down under 110 mb so i have to use multistage build

our current dockerfile uses single-stage build

in multi-stage build
1.stage 1 (build): install dependencies,run build-> produces dist/
2.stage 2(production): copy only dist/ into a tiny nginx:alpine image

so i amde a dist/ from stage 1 and use it in stage 2 and just give runtime space to it

and then rebuild the dockerfile by docker compose up --build -d
and checked the image size and here comes the magic that is i decreased to 45 mb

now i head towards phase 3

first i went straight towards docker-compose.yml
in this file i saw that PostSQL databse is not hardcoded it has environment variables set to
database:
  environment:
    POSTGRES_USER: postgres          # ← env var, not hardcoded in code
    POSTGRES_PASSWORD: postgrespassword
    POSTGRES_DB: orbis

The backend reads the connection string from an env var too (line 22):

yaml

DATABASE_URL: postgresql://postgres:postgrespassword@database:5432/orbis?schema=public

for 2nd thing the port should not be exposed publically for database The database service has no ports: mapping
database:
 NO "ports:" section = not exposed to host
networks:
  -backend-network    # Only accessible within backend-network

Only the proxy service exposes port 80 to the host. The database is only reachable by containers on backend-network.

now for 3 rd thing we need docker volume its already done
database:
  volumes:
    - postgres_data:/var/lib/postgresql/data    # named volume
volumes:
  postgres_data:    # declared at bottom

now 4 th requirement is i need to prove that data survives docker compose down +docker compose up

so i made sure first all the containers are running
then i inserted test records and got immediate output as
CREATE TABLE
INSERT 0 1
INSERT 0 1

then i ran the command
docker exec -it orbis-db psql -U postgres -d orbis -c "SELECT * FROM test_persistence;"
what is basically a sql query to print evry data from test persistence table

 id |           message           |         created_at
----+-----------------------------+----------------------------
  1 | Phase 3 persistence test    | 2026-10-03 04:27:10.964063
  2 | Data should survive restart | 2026-10-03 04:27:10.964063
(2 rows)

then i destroyed the containers by running docker compose down and bring all the conatiners back with docker compose up -d

and started the query (docker exec -it orbis-db psql -U postgres -d orbis -c "SELECT * FROM test_persistence;) again
and guess those 2 rows are const output

 id |           message           |         created_at
----+-----------------------------+----------------------------
  1 | Phase 3 persistence test    | 2026-10-03 04:27:10.964063
  2 | Data should survive restart | 2026-10-03 04:27:10.964063
(2 rows)

The data survived because the database files are stored in a named Docker volume (postgres_data) mounted at /var/lib/postgresql/data, not inside the container itself. When I ran docker compose down, the container was deleted but the volume stayed on the host. On docker compose up, the new container reattached to the same volume, so PostgreSQL found its existing files and the test record was still there

so here i proved that data survived through docker compose down and docker compose up (we cannot run docker compose down -v) because it removes containers,networks,and volumes(data lost)

with this i completed my phase 3 as well

now i am heading towards my phase 4 its bascially about docker networking

Configure Docker networking to implement proper network segmentation:

1.Frontend and reverse proxy sit on one network
2.Backend and database sit on a separate, isolated network that only the backend can reach
3.Only the reverse proxy is exposed to the host on ports 80 and 443
4.No other service ports are exposed to the host

at last our submission requirements are-

1.docker network inspect output for each defined network
2.Verification tests proving backend/database are unreachable from outside their network, but internal communication works via docker compose exec

first of all what our current setup is
frontend-network
orbis-proxy  ←→  orbis-backend  ←→  orbis-frontend

backend-network
orbis-proxy  ←→  orbis-backend  ←→  orbis-db

so basically orbis proxy is on backend-network but proxy should not have direct access to the database

orbis-backend is on frontend-network — this is actually needed so proxy can route /api/ to backend

No port 443 exposed on proxy

so i will target to make it something like this

frontend-network
orbis-proxy ←→ orbis-frontend ←→ orbis-backend  
↕ (host:80,443)

backend-network
orbis-backend ←→ orbis-db

first i changed the proxy service and removed backend-network and add port 443
prt 443 is the standard,globally recognized TCP port assigned by the Internet Assigned Numbers Authority (IANA) and for secure web traffic (HTTPS) and port 80 is for HTTP traffic. i didnt know anything about this i just googled it and found it out

now i rebuild

and then i verify network membership

![Frontend Network Inspection](./photo-of-submission.md/docker-network-inspect-frontend.png)

![Frontend Network Inspection](./photo-of-submission.md/docker-network-inspect-frontend-extended.png)

![Backend Network Inspection](./photo-of-submission.md/docker-network-inspect-backend.png)

![Backend Network Inspection](./photo-of-submission.md/docker-network-inspect-backend-extended.png)

now i will verify that backend and database are unreachable from outside their network, but internal communication works via docker compose exec

![Verification Test](./photo-of-submission.md/verification-isolation.png)

so here i ran 6 tests to verify the isolation of backend and database from outside their network and also verified that internal communication works via docker compose exec

Test 1: Proxy CANNOT reach database directly
Test 2: Frontend CANNOT reach database
Test 3: Backend CAN reach database (internal communication works)
Test 4: Backend CAN be reached by proxy (API routing works)
Test 5: Host CANNOT reach database port directly
Test 6: Host CAN reach proxy

now i am heading towards phase 5 which is about Nginx SSl termination and HTTPS redirection

first requirement is to generate and configure self-signed SSL certificate for Nginx or Let's Encrpyt/Certbot SSL certificate

so i made nginx directory in project and ssl forlder inside it and then i generated self-signed SSL certificate using openssl command

openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout nginx/ssl/server.key \
  -out nginx/ssl/server.crt \
  -subj "/C=IN/ST=Karnataka/L=Mangalore/O=NITK/CN=localhost"

second requirement is to terminate SSL on port 443
3 rd requirement is to proxy decrypted traffic to backend and frontend services
4 th requirement is automatically redirect all HTTP port 80 traffic to HTTPS

i changed nginx.conf file to reflect these changes and i added 1 more line in docker compose file to add SSL volume mount to nginx service
volumes:
  - ./nginx/ssl:/etc/nginx/ssl:ro

then i rebuild the dockerfile and checked the webpage and it is working fine with https and also redirecting http to https

Test 1: HTTP->HTTPS redirection works
test 2: HTTPS works
test 3 : https API routing works

while i was running test 3 an error occured

![error in phase](./photo-of-submission.md/error-while-phase-5.png)

so i was like i got fucked now but see i am this task is basically a devops task so i have to deal with it so i took the help from ai found out the error
as i run below command
 docker compose exec backend sh -c "nc -zv database 5432"

it can open databse but still test 3 was failing because there was already an existing table in the databse which i created during phase 3 test that is test persistence table and i got to know this because i searched P2021 on ai what does this error message signify
the logs showed the DB connection worked but the schema had no tables. Running migrate deploy gave P3005 because a leftover test table made the schema non-empty
Fix: removed the test table, ran prisma migrate deploy, and confirmed the API response.
then test 3 also worked fine

now iam heading towards my phase 6: Consolidated Orchestration and Build Cache Analysis

we need to fulfill 5 requirments
1-Finalize docker-compose.yml for single-command startup (docker compose up --build)
2-Extract inline/hardcoded config values and secrets into .env file
3-Modify a single line of application source code and rebuild
4-Analyze build log: which layers got cache hits vs rebuilds
5-Detail how Dockerfile instruction ordering influences caching efficiency

so now i will create .env file in same directory of docker-compose.yml becuase
Docker Compose automatically reads a file named .env in the same folder as docker-compose.yml

main advantage of maintaining .env file is  
Secrets stay out of the compose file, so you can share the compose file without sharing passwords.
One place to change settings, instead of hunting through the file 
then i checked the docker compose compose file by bash cmd
docker comose config and it is taking value from .env file

Cache exp
I first built everything once so all layers were cached. Then I changed exactly one line in backend/server.js:

- console.log(`Server running on port ${PORT}`);
+ console.log(`Orbis Server running on port ${PORT}`);

and rebuilt with
docker compose build --progress=plain backend 2>&1 | tee build_log.txt

now before changing the line in server.js

Step                             Result   Why
FROM node:20-alpine              CACHED   base image unchanged
RUN apk add --no-cache openssl   CACHED   command unchanged
WORKDIR /app                     CACHED   unchanged
COPY . .                         REBUILT  server.js changed, so this layer's contents changed
RUN npm ci                       REBUILT  comes after a changed layer, so it is invalidated even though dependencies did not change
RUN npx prisma generate          REBUILT  same reason

after changing the line

Step                            Result  Why
FROM, apk add openssl, WORKDIR  CACHED  unchanged
COPY package*.json ./           CACHED  package files did not change
RUN npm ci                      CACHED  its inputs did not change
COPY prisma ./prisma            CACHED  schema did not change
RUN npx prisma generate         CACHED  its inputs did not change
COPY . .                        REBUILT server.js changed

Frontend: i did not touch the frontend in this rebuild, so every frontend layer, including npm install and npm run build, stayed cached. The frontend Dockerfile already copies package.json and package-lock.json and runs npm install before copying the source, so a source change there would only rerun the source copy and the build step, not the dependency install

i learnt after this,docker saves every Dockerfile instruction as a layer. On the next build it checks each instruction in order. If the instruction and the files it uses are unchanged, it reuses the old layer (CACHED). The important rule is that once one layer changes, every layer after it is rebuilt too, even if those later steps did not change themselves.

That is why instruction order matters. Things that rarely change (base image, system packages, package.json, npm install) should come first, and things that change constantly (my own source code) should come last. Then an everyday code change only rebuilds the last couple of steps instead of reinstalling every dependency.

Blank white page. The page at https://localhost was empty even though the HTML, JS and CSS were all served successfully. The browser console showed Uncaught Error: supabaseUrl is required. The frontend creates a Supabase client on load, and its URL was undefined. Vite only reads VITE_ variables at build time, and my Docker build never passed them, so they were missing from the compiled JavaScript. I fixed it by declaring ARG/ENV for VITE_SUPABASE_URL and VITE_SUPABASE_ANON_KEY before npm run build, passing them through build.args in the compose file from .env, and rebuilding with --no-cache. I used dummy values, so features that need a real Supabase project will not work, but the page now renders.

i didnt know much about phase 7 so i headed towards phase 8 (bonus) Container Hardening and CI Pipeline

Non-root users

in bakcend i created a system user and group in the Dockerfile and switched to it with `USER appuser` after `npm ci` and `prisma generate`. Source files are copied with `COPY --chown`, so no extra `chown -R` layer doubles the image size. The app listens on port 4000, which needs no root privileges

in frontend the final stage now uses `nginxinc/nginx-unprivileged:alpine`. The stock `nginx:alpine` image can't run as a non-root user, because its PID file and cache directories are root-owned. The unprivileged image is built for this and listens on **8080**, so I changed the proxy upstream in `nginx/default.conf` from `frontend:80` to `frontend:8080`. The build stage still runs as root, but it is discarded and never shipped.

 docker compose exec backend id
uid=100(appuser) gid=101(appgroup) groups=101(appgroup)

docker compose exec frontend id
uid=101(nginx) gid=101(nginx) groups=101(nginx)

I added `backend/.dockerignore` and `frontend/.dockerignore` excluding `node_modules`, `.env` and `.env.*`, `.git`, `dist` (frontend), logs, and docs. The point is:

 docker compose exec backend ls -a /app
.                  node_modules       server.js
..                 package-lock.json  src
.dockerignore      package.json
database           prisma

- The frontend is multi-stage: Node, npm and `node_modules` exist only in the build stage. The runtime image has only Nginx and the compiled static files. Final size: [PASTE docker images output].
- The backend installs only `openssl`, which Prisma's engine needs, with `--no-cache` so the package index isn't kept. No compilers or extra tools are installed.
- I did not remove `openssl` from the runtime image because Prisma fails without it (the `libssl` error from Phase 1).

File: `.github/workflows/docker-build.yml`. It triggers on pushes and pull requests to `main`/`master` and runs two jobs:
1. **lint-dockerfiles:** runs hadolint on both Dockerfiles. I ignore `DL3018` (pinning exact `apk` package versions) because pinned Alpine versions break when the package index moves on.
2. **build-images:** builds both images with Buildx (`push: false`, `load: true` so the image is available locally), uses the GitHub Actions layer cache, checks that both containers run as non-root, and fails if the frontend image exceeds 110 MB. Frontend `VITE_*` build args use placeholder values; no real secrets are in the workflow.

**Result:** [PASTE link to the successful Actions run and/or a screenshot of the green check]

### Use of AI assistance

I used an AI assistant to draft the initial hardened Dockerfiles and workflow. I then reviewed them, tested them locally, and fixed the problems I found (the Nginx non-root issue, the `load: true` flag so the CI size check can see the image, and the hadolint threshold). I can explain every line.

### What I'd improve next

- Pin base images to exact versions or digests.
- Add a vulnerability scan (for example Trivy) to the CI.
- The proxy container still starts as root so it can bind port 443 and read the TLS certificate; it drops to the `nginx` user for its worker processes. A fully non-root proxy would use the unprivileged image on high ports.