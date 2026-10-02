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

