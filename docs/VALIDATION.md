# Verification & Validation Evidence

This document contains the captured terminal evidence demonstrating the reproduction of the configuration failure, diagnosis of baked secrets, and the clean startup verification following the fix.

---

## 1. Before Fix: Failure Reproduction & Diagnosis

### A. Docker Build Warning (Secret Detected in Layer)
```console
$ docker build -t envlab .
#0 building with "desktop-linux" instance using docker driver
#1 [internal] load build definition from Dockerfile
#1 DONE 0.1s
...
#11 naming to docker.io/library/envlab:latest done

 1 warning found (use docker --debug to expand):
 - SecretsUsedInArgOrEnv: Do not use ARG or ENV instructions for sensitive data (ENV "API_KEY") (line 11)
```

### B. Container Crash on Startup
```console
$ docker run --name app envlab
app

> env-config-lab@1.0.0 start
> node server.js

Error: DATABASE_URL is undefined
```
*(Process terminated with exit code 1)*

### C. Container Logs (`docker logs app`)
```console
$ docker logs app

> env-config-lab@1.0.0 start
> node server.js

Error: DATABASE_URL is undefined
```

### D. Attempting `docker exec app printenv` on Crashed Container
```console
$ docker exec app printenv
Error response from daemon: container ccdb08bf37026e3232eaf8f38c00f1e14e7b1ebad0184e00e7693fd0341a024c is not running
```

### E. Inspecting Built-in Environment (`docker run --rm envlab printenv`)
```console
$ docker run --rm envlab printenv
NODE_VERSION=18.20.8
HOSTNAME=8ae326de66bc
YARN_VERSION=1.22.22
SHLVL=1
HOME=/root
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
LOGLEVEL=info
API_KEY=super-secret-key
PWD=/usr/src/app
```
**Diagnosis**:
- `API_KEY=super-secret-key`: Plaintext secret hardcoded directly into the Docker image.
- `LOGLEVEL=info`: Typo in variable name (application expects `LOG_LEVEL` with an underscore).
- `DATABASE_URL`: Completely missing, triggering startup crash.

### F. Image Layer Analysis (`docker history envlab`)
```console
$ docker history envlab
IMAGE          CREATED          CREATED BY                                      SIZE      COMMENT
ef082489e01b   14 minutes ago   CMD ["npm" "start"]                             0B        buildkit.dockerfile.v0
<missing>      14 minutes ago   ENV LOGLEVEL=info                               0B        buildkit.dockerfile.v0
<missing>      14 minutes ago   ENV API_KEY=super-secret-key                    0B        buildkit.dockerfile.v0
<missing>      14 minutes ago   COPY app/ . # buildkit                          24.6kB    buildkit.dockerfile.v0
<missing>      14 minutes ago   RUN /bin/sh -c npm install # buildkit           12.9MB    buildkit.dockerfile.v0
<missing>      14 minutes ago   COPY app/package*.json ./ # buildkit            20.5kB    buildkit.dockerfile.v0
<missing>      11 days ago      WORKDIR /usr/src/app                            16.4kB    buildkit.dockerfile.v0
```

---

## 2. After Fix: Clean Startup & Verification

### A. Clean Image Build (No Security Warnings)
```console
$ docker build -t envlab .
#0 building with "desktop-linux" instance using docker driver
#1 [internal] load build definition from Dockerfile
#1 DONE 0.2s
...
#11 naming to docker.io/library/envlab:latest done
```
*(Build completed cleanly with 0 warnings)*

### B. Clean Layer History (`docker history envlab`)
```console
$ docker history envlab
IMAGE          CREATED          CREATED BY                                      SIZE      COMMENT
cef4f7ad1a09   17 minutes ago   CMD ["npm" "start"]                             0B        buildkit.dockerfile.v0
<missing>      17 minutes ago   COPY app/ . # buildkit                          24.6kB    buildkit.dockerfile.v0
<missing>      17 minutes ago   RUN /bin/sh -c npm install # buildkit           12.9MB    buildkit.dockerfile.v0
<missing>      17 minutes ago   COPY app/package*.json ./ # buildkit            20.5kB    buildkit.dockerfile.v0
<missing>      11 days ago      WORKDIR /usr/src/app                            16.4kB    buildkit.dockerfile.v0
```
*(Neither `API_KEY` nor `LOGLEVEL` are present in the image)*

### C. Clean Container Run with `--env-file` and `-e API_KEY`
```console
# Local git-ignored .env supplies non-secret configuration:
# DATABASE_URL=postgres://user:password@localhost:5432/mydb
# LOG_LEVEL=info

$ docker run -d --env-file .env -e API_KEY="runtime-secret-key-12345" --name app-fixed -p 3000:3000 envlab
5840bd0cf6675f278f8c73f2585bf7441f6095725cecb1b889a659632d816a76
```

### D. Clean Startup Logs (`docker logs app-fixed`)
```console
$ docker logs app-fixed

> env-config-lab@1.0.0 start
> node server.js

Starting app with log level: info
Server is running on port 3000
```

### E. Verified Injected Variables (`docker exec app-fixed printenv`)
```console
$ docker exec app-fixed printenv
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=5840bd0cf667
DATABASE_URL=postgres://user:password@localhost:5432/mydb
LOG_LEVEL=info
API_KEY=runtime-secret-key-12345
NODE_VERSION=18.20.8
YARN_VERSION=1.22.22
HOME=/root
```

### F. HTTP Health Check (`curl http://localhost:3000/health`)
```console
$ curl http://localhost:3000/health
{"status":"ok"}
```
