# Root Cause Analysis

Fill this in as you go. Bullet points are fine — be specific, not long.

---

## Defect 1: Proxy image fails to build

### Symptom / error output

Running `./scripts/run-tests.sh` failed during Nginx installation in the proxy image:

```text
Configuration file '/etc/nginx/nginx.conf'
 ==> File on system created by you or by a script.
 ==> File also in package provided by package maintainer.
*** nginx.conf (Y/I/N/O/D/Z) [default=N] ? dpkg: error processing package nginx-common (--configure):
 end of file on stdin at conffile prompt
```

### How I diagnosed it

The build output showed that `dpkg` asked whether to keep the existing `nginx.conf` or replace it with the package's version. The prompt failed because the build could not provide an answer.

I reviewed `proxy/Dockerfile` and found that the custom configuration was copied into the image before Nginx was installed.

### Root cause

The Dockerfile copied the custom configuration to `/etc/nginx/nginx.conf` before installing Nginx. During installation, `dpkg` detected the existing file and asked which version to keep. Since the Docker build runs non-interactively, it could not answer the prompt.

### Fix

I reversed the order of the Dockerfile instructions:

1. Install Nginx and the required packages using `apt-get`.
2. Copy the custom `nginx.conf` into `/etc/nginx/nginx.conf`.

This allows Nginx to install successfully before its default configuration is replaced with the custom configuration.

---

## Defect 2: Proxy cannot reach the backend

### Symptom / error output

After fixing the proxy image build, running `./scripts/run-tests.sh` again failed with the following error:

```text
TEST RUNNER CRASHED: HTTP Error 502: Bad Gateway

[error] 7#7: *2 connect() failed (111: Connection refused) while connecting to upstream,
request: "GET /api/info HTTP/1.1",
upstream: "http://127.0.0.1:8080/api/info",
host: "proxy:80"
```

### How I diagnosed it

The logs showed that Nginx was trying to forward requests to `127.0.0.1:8080`.

I checked `docker-compose.yml` and found that the backend service is named `backend` and listens on port `8080`. However, the address in `proxy/nginx.conf` pointed to the proxy container itself.

### Root cause

Each container has its own network namespace. Inside the proxy container, `127.0.0.1` refers to the proxy itself, not the backend.

Nothing was listening on port 8080 inside the proxy container, so the connection was refused and Nginx returned 502 Bad Gateway.

### Fix

I updated the upstream configuration in `proxy/nginx.conf` to use the backend service name:

```nginx
upstream app_backend {
    server backend:8080;
}
```

Docker Compose lets containers on the same network find each other by service name. Using `backend:8080` allows Nginx to forward requests to the backend container.

---

## Defect 3: Tests still fail after fixing defect 2

### Symptom / error output

After fixing the upstream address, the tests still failed with the following error:

```text
[error] 7#7: *2 connect() failed (111: Connection refused) while connecting to upstream,
client: 172.18.0.4, server: localhost,
request: "GET /api/info HTTP/1.1", upstream: "http://172.18.0.2:8080/api/info",
host: "proxy:80"
```

Nginx was now trying to connect to the backend container, but the connection to port 8080 was still refused.

### How I diagnosed it

I reviewed `backend/app.py` and found that the backend loads four configuration values before starting its HTTP server. Each value takes two seconds to load by default, so the backend needs about eight seconds before it can answer requests.

The backend already has a `/health` endpoint in `backend/app.py`, and the proxy has a `/nginx-health` endpoint in `proxy/nginx.conf`. These endpoints can check whether each service is ready to answer HTTP requests.

In `docker-compose.yml`, the original `depends_on` settings started the containers in order, but did not wait for the applications inside them to answer requests.

### Root cause

The tests sent requests before the backend finished loading its configuration. The backend container was running, but nothing was listening on port 8080 yet. This caused Nginx to return 502 Bad Gateway.

The tests could also start before Nginx was ready to answer requests.

### Fix, and why I chose it over the alternatives

I added health checks to `docker-compose.yml`:

1. Check the backend's `/health` endpoint using Python. Start the proxy only after this check succeeds.
2. Check the proxy's `/nginx-health` endpoint using curl. Start the test-runner only after this check succeeds.

The `condition: service_healthy` setting in `depends_on` makes Compose wait for each health check to pass.

I chose health checks instead of a fixed sleep because startup can take longer on a slower machine. Health checks wait until each service can answer requests, with a retry limit so they do not wait forever. The backend code and tests remain unchanged.

Verification included `docker compose config --quiet` and a successful test run through WSL. Both integration tests passed, cleanup completed, and the test script exited with code 0.

---

## Defect 4: CI pipeline fails

### Symptom / error output

The CI step `Execute test suite` failed with the following error:

```text
./scripts/run-tests.sh: Permission denied
```

### How I diagnosed it

I reviewed `.github/workflows/ci.yml` and found that it runs `./scripts/run-tests.sh` directly. This requires the script to have executable permission.

### Root cause
The script did not have executable permission, so the CI could not run it directly.
The proxy build failure, incorrect upstream address, and missing startup readiness checks also prevented the test suite from passing.

### Fix

I added executable permission to `scripts/run-tests.sh` so it can run directly. This permission must be recorded in Git so it is preserved when the CI checks out the repository.

The fixes from Defects 1-3 resolved the image build, backend routing, and startup readiness problems. After pushing these changes, the CI workflow passed. No change to the workflow file was needed.

---

## Bonus questions

### 1. Reduce the proxy image size

I replaced the `ubuntu:24.04` base image with `nginx:stable-alpine-slim`. This image already includes Nginx and uses a smaller Alpine-slim Linux base.

The proxy does not use `htop`, `vim`, or `git` at runtime, so these packages are no longer installed. 

The proxy health check still needs `curl`, so the Dockerfile installs it using `apk add --no-cache curl`. Alpine uses `apk` instead of Ubuntu's `apt-get`.

The custom `nginx.conf` is still copied into `/etc/nginx/nginx.conf`.

#### Image size comparison

| Proxy image | Disk usage | Content size |
|---|---:|---:|
| Ubuntu-based | 443 MB | 121 MB |
| Alpine-Slim-based | 37.1 MB | 10.7 MB |

I used `docker image ls f5-devops-challenge-proxy` to record both sizes. 

#### Verification

I ran `./scripts/run-tests.sh` to confirm that the proxy health check and backend forwarding still work with the new base image.

---

## Anything you would do differently with more time
