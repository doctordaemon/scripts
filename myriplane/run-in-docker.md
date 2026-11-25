# Running myriplane in Docker from this checkout

Run everything from the repo folder:
```bash
cd ~/myriplane-tenant-isolation-poc/myriplane
```

## How it fits together

- **Your machine** builds two programs: `myriplane` (the server) and `myrictl` (the command-line client).
- **Docker** packs the `myriplane` binary into an image and runs three containers:
  - `cert-manager` runs once at startup and creates the TLS certificates.
  - `myriplane-etcd` is the database.
  - `myriplane` is the server, on `https://localhost:443`.
- **You** use `./myrictl` from your machine to talk to the server.

## First-time setup (or after a long break)

### 1. Use a clean terminal and load the build tools
```bash
env | grep MYRI                              # should print nothing
unset $(env | grep -o '^MYRI[A-Z_]*')        # only if it printed something
source env.sh
```
- **Why the `MYRI` check:** the non-Docker setup's file (`hack/etcd-dev-etc/myriplane-env.sh`) sets `MYRIPLANE_*` variables that point everything at port 5555. Those variables override `~/.myri/myrictl.yaml`, so `myrictl` would try 5555 instead of the Docker server on 443. Never source that file in a terminal you use with Docker.
- **Why `source env.sh`:** `setup-dev.sh` installed Go, protoc and the protoc plugins into the repo's `.bin/` folder, but your terminal doesn't know about that folder. A script can't change your terminal's settings, so you have to load `env.sh` yourself. Do this in **every new terminal** before building. It's safe to use with Docker because it only adds the build tools to PATH.

### 2. Build the binaries
```bash
make local
```
This builds `./myriplane` and `./myrictl` in the repo folder. The Docker image doesn't compile anything. It just copies these files in, so you must build them first.

### 3. Create two files the Docker build expects
```bash
mkdir -p registry-data
touch myrictl.yaml
```
The Dockerfile copies both of these, and git ignores them, so a fresh checkout doesn't have them.
- `registry-data/` holds container images for the server's built-in registry. Empty is fine.
- `myrictl.yaml` is only a placeholder. Leave it empty, because a real one contains your login token and would end up inside the image.

You only need to do this once per checkout.

### 4. Stop the old stack, if it's running
```bash
docker inspect myriplane --format '{{index .Config.Labels "com.docker.compose.project.working_dir"}}'
```
This prints the folder the running `myriplane` container was started from. If it's a different folder, stop that stack:
```bash
(cd <that-folder> && docker compose down)
```
Compose names a stack after its folder. Both your checkouts are in folders called `myriplane`, so they share the same containers and data, and only one can run at a time.

### 5. Build the image and start the stack
```bash
docker compose up -d --build
docker compose ps
```
`--build` puts your newly built binaries into the image, and `-d` runs it in the background. In `docker compose ps` you should see `cert-manager` exited with code 0, `etcd` healthy and `myriplane` up.

### 6. Set up myrictl (once, and again after certs are regenerated)
```bash
docker cp myriplane:/certs/ca.crt ./myriplane-docker-ca.crt
```
The server's certificate is signed by a private CA that was created inside Docker. `myrictl` needs a copy of that CA to trust the server.

Then edit `~/.myri/myrictl.yaml` so it contains:
```yaml
api_url: "https://localhost:443"
apiserver_cert: "/home/yash/myriplane-tenant-isolation-poc/myriplane/myriplane-docker-ca.crt"
log_level: "info"
```
This tells `myrictl` where the server is and which CA to trust, so you don't have to pass flags every time. The `token_admin:` line from the old setup can stay; logging in replaces it.

### 7. Log in
```bash
./myrictl login admin      # default password: admin
```

## Everyday commands

| I want to… | Command |
|---|---|
| Rebuild after changing code | `source env.sh && make local && docker compose up -d --build myriplane` |
| Check it's running | `docker compose ps` |
| See server logs | `docker compose logs -f --tail 100 myriplane` |
| Stop it (keeps data) | `docker compose down` |
| Start it again | `docker compose up -d` |
| Delete everything and start empty | `docker compose down -v` (deletes the database and certs) |

## When it breaks

**Certificates expire after 30 days.** The current ones expire on **Dec 25 2026**. The `cert-manager` container only creates certs when there are none, so it never replaces expired ones. That's what broke the old stack. The symptoms are `myriplane` restarting over and over and `certificate has expired` in the logs.

Check the expiry date:
```bash
docker run --rm -v myriplane_certs:/certs:ro alpine:3.24.1 cat /certs/etcd/server.crt | openssl x509 -noout -enddate
```
Renew the certs (your data is kept):
```bash
docker compose down
docker volume rm myriplane_certs
docker compose up -d
```
Then redo step 6 (copy the CA again, because it's new) and step 7.

**`protoc-gen-go: program not found`** means you skipped `source env.sh` in this terminal.

**`dial tcp [::1]:5555: connection refused`** (often with `rpc error`) means your terminal has `MYRIPLANE_*` / `MYRICTL_*` variables from the non-Docker setup. Clear them with `unset $(env | grep -o '^MYRI[A-Z_]*')` or open a new terminal.

**`unknown flag: --apiserver-cert`** means the flag name is wrong. It's `--api-cert`. With step 6 done, you don't need the flag at all.

**`x509: certificate signed by unknown authority`** means `myrictl` doesn't have the current CA. Redo step 6.

## Other things to know
- `make local` regenerates the `.pb.go` files, and some may show as changed in `git status` because the protoc version differs. Don't commit them unless you mean to.
- The local non-Docker setup (`hack/run-etcd-dev-ssl-peer.sh`, port 5555) is a separate way to run myriplane. Keep it in its own terminal.
