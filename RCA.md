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

### How I diagnosed it

### Root cause

### Fix

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
