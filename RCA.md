# Root Cause Analysis

Fill this in as you go. Bullet points are fine — be specific, not long.

---

## Defect 1: Proxy image fails to build

### Symptom / error output

I ran `./scripts/run-tests.sh` and noticed that the proxy Docker image failed to build with the following error:

```text
failed to solve: process "/bin/sh -c apt-get update && apt-get install -y nginx curl htop vim git" did not complete successfully: exit code: 100
```

### How I diagnosed it

I inspected the Docker build output and reviewed the proxy Dockerfile. I noticed that the custom `nginx.conf` file was being copied into the image before Nginx was installed.

### Root cause

The Dockerfile copied the custom configuration file to `/etc/nginx/nginx.conf` before installing Nginx. 
During installation, `dpkg` detected that a configuration file already existed and asks which version to keep. Since the Docker build runs non-interactively it cannot answer the prompt.

### Fix

I reversed the order of the Dockerfile instructions:
1. Install Nginx and the required packages using apt-get.
2. Copy the custom nginx.conf file into /etc/nginx/nginx.conf.
This allows Nginx to install successfully before its default configuration is replaced with the custom configuration.

---

## Defect 2: Proxy cannot reach the backend

### Symptom / error output

After fixing the proxy image build, I ran `./scripts/run-tests.sh` again, but the integration tests failed with the following error:

```text
TEST RUNNER CRASHED: HTTP Error 502: Bad Gateway

[error] 7#7: *2 connect() failed (111: Connection refused) while connecting to upstream,
request: "GET /api/info HTTP/1.1",
upstream: "http://127.0.0.1:8080/api/info",
host: "proxy:80"
```

### How I diagnosed it

The logs showed that Nginx was trying to forward requests to `127.0.0.1:8080`.
I checked docker-compose.yml and found that the backend service is named backend and listens on port 8080. However, Nginx was configured to forward requests to 127.0.0.1:8080, the local address which points to the proxy container itself.

### Root cause

Each Docker container has its own network namespace, and since Nginx runs in a separate container from the backend, forwarding requests to `127.0.0.1:8080` caused Nginx to connect to itself rather than the backend.
Nothing was listening on port 8080 inside the proxy container, resulting in a connection refusal and an HTTP 502 Bad Gateway response.

### Fix

I updated the upstream configuration in `nginx.conf` to use the backend service name:
```nginx
upstream app_backend {
    server backend:8080;
}
```
Docker Compose provides internal DNS resolution, allowing containers on the same network to communicate using their service names.
By using `backend:8080` instead of `127.0.0.1:8080`, Nginx can correctly forward requests to the backend container.
---

## Defect 3: Tests still fail after fixing defect 2

### Symptom / error output

### How I diagnosed it

### Root cause

### Fix, and why I chose it over the alternatives

---

## Defect 4: CI pipeline fails

### Symptom / error output

### How I diagnosed it

### Root cause

### Fix

---

## Bonus questions

Optional. Answer only the ones you got to — number them, skip the rest.

>

---

## Anything you would do differently with more time
