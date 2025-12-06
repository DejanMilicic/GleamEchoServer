# Deployment Guide

This deployment guide describes how to set up your Gleam application on Hetzner server

## Tech Stack

- local dev: macOS
- hosting: Hetzner
- host OS: Ubuntu
- source code repo: GitHub
- CI/CD: GitHub Actions
- OCI image
- Podman

TODO: explain rationale for every choice

## Set up local SSH keys

TODO. 
key will be automatically used every time use ssh into your server

## Provision Hetzner server

- https://console.hetzner.com/
- enter project in which you want to create a server
- left menu: Servers
- "Add Server"
- select Type and Location
- Image: Ubuntu
- Networking: check if your dev machine had IPv6. If it does not, you need to select IPv4 too
- SSH keys: add your public key
- "Create & Buy now"

## Set up ssh locally

Add entry to `~/.ssh/config`

```
Host [HOST_NAME]
	HostName [IPv4]
	User root
	Port 22
```

Access server via `ssh [HOST_NAME]`

## Hardening server

1. Update everything

```
apt update && apt full-upgrade -y
apt autoremove -y
```

then 
```
reboot
```

2. Enable automatic security updates

```
apt install -y unattended-upgrades
dpkg-reconfigure -plow unattended-upgrades
```

3. Secure SSH (`/etc/ssh/sshd_config`)

```
PermitRootLogin prohibit-password
PasswordAuthentication no
PubkeyAuthentication yes
MaxAuthTries 3
```

then
```
systemctl restart ssh
```

4. Configure Firewall

```
ufw default deny incoming
ufw default allow outgoing
ufw allow 22
ufw allow 80
ufw allow 443
ufw enable
```

you can inspect configuration with `ufw status verbose`

5. Install `fail2ban`

```
apt install -y fail2ban
```

Add following to `/etc/fail2ban/jail.local`

```
[DEFAULT]
allowipv6 = auto

[sshd]
enabled = true
maxretry = 3
bantime = 1h
```

then

```
systemctl enable fail2ban
systemctl start fail2ban
```

6. Kernel hardening (`/etc/sysctl.conf`)

```
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
```

Then 

```
sysctl -p
```

## Prepare your application for Production Deployment

1. Ensure your application is listening on `0.0.0.0`

```gleam
    // Start the web server process
    let assert Ok(_) =
        wisp_mist.handler(router.handle_request, secret_key_base)
        |> mist.new
        |> mist.bind("0.0.0.0") // <- add this line
        |> mist.port(8000)      // <- rest of this guide assumes port 8000 is used
        |> mist.start_http
```

2. Create a health check script

Create a file named `healthcheck.sh` with these contents:

```
#!/bin/sh
#
# This script is run periodically by the Podman container engine to check if
# the application is healthy. If the application instance is determined to be
# unhealthy then Podman will terminate the container, causing systemd to
# replace it with a new instance.
#
# If this script returns exit code 0 then the check is a pass.
# Any other exit code is a failure, with multiple failures in a row meaning the
# instance is unhealthy.
#
# wget is used to send a HTTP request to the application, to check it is
# serving traffic.
# You may choose to add additional health check logic to this script.
#

exec wget --spider --quiet 'http://127.0.0.1:8000'
```

3. Create a `Dockerfile`

```dockerfile
ARG ERLANG_VERSION=28.2.0.0
ARG GLEAM_VERSION=v1.13.0

# Gleam stage
FROM ghcr.io/gleam-lang/gleam:${GLEAM_VERSION}-scratch AS gleam

# Build stage
FROM erlang:${ERLANG_VERSION}-alpine AS build
COPY --from=gleam /bin/gleam /bin/gleam
COPY . /app/
RUN cd /app && gleam export erlang-shipment

# Final stage
FROM erlang:${ERLANG_VERSION}-alpine
ARG GIT_SHA
ARG BUILD_TIME
ENV GIT_SHA=${GIT_SHA}
ENV BUILD_TIME=${BUILD_TIME}
COPY healthcheck.sh /app/healthcheck.sh
RUN \
  chmod +x /app/healthcheck.sh \
  && addgroup --system webapp \
  && adduser --system webapp -g webapp
COPY --from=build /app/build/erlang-shipment /app
WORKDIR /app
ENTRYPOINT ["/app/entrypoint.sh"]
CMD ["run"]
```

You can use `brew list --versions gleam erlang` to get versions of Gleam and Erlang installed on your machine

4. Build your container on CI

Create a file at `.github/workflows/build-container.yml` with these contents:

````yaml
name: Build container image
on:
  push:
    tags:
      - v*

jobs:
  push:
    runs-on: ubuntu-latest
    permissions:
      packages: write
      contents: read
    steps:
      - uses: actions/checkout@v4

      - name: Build image
        run: docker build . --file Dockerfile --tag webapp

      - name: Log in to registry
        run: echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io -u ${{ github.actor }} --password-stdin

      - name: Push image
        run: |
          IMAGE_ID=ghcr.io/[LOWERCASED_GITHUB_USERNAME]/[LOWERCASED_REPO_NAME]
          TAG="$IMAGE_ID":$(echo "${{ github.ref }}" | sed -e 's,.*/\(.*\),\1,')
          docker tag webapp "$TAG"
          docker push "$TAG"
````

We will be using GitHub actions to build and publish the container image to the GitHub container registry using docker each time a git tag starting with `v` is pushed to the repo. For example, `v1.0.0`.

After you have pushed these changes push a new git tag to GitHub. This will trigger the workflow, which you can see in your GitHub repo’s “Actions” tab.

```bash
git tag v_test
git push --tags
```

## Prepare host for your application

1. Install Caddy

```
apt install --yes caddy
```

2. Install Podman

```
apt install --yes podman
```

3. Define your Podman container

If you are using a private GitHub repository you will need create a GitHub personal access token with `read:packages` permissions in the GitHub security settings, and then use it to log in on the server.

```
echo "YOUR_GITHUB_PAT" | podman login ghcr.io -u YOUR_GITHUB_USERNAME --password-stdin
```

Create a Podman systemd container file

```
nano /etc/containers/systemd/webapp.container
```

with these contents

````yaml
[Unit]
Description=My Gleam web application
After=local-fs.target

[Container]
Image=ghcr.io/[LOWERCASED_GITHUB_USERNAME]/[LOWERCASED_REPO_NAME]:latest
AutoUpdate=registry

# Expose the port the app is listening on
PublishPort=8000:8000

# Restart the service if the health check fails
HealthCmd=sh -c /app/healthcheck.sh
HealthInterval=30s
HealthTimeout=5s
HealthRetries=3
HealthOnFailure=restart

[Install]
WantedBy=multi-user.target default.target
````

`AutoUpdate=registry` will instruct podman to periodically check for new version of container. 

Alternatively, you can manually pull new version of a container

```
podman auto-update
```

systemctl daemon-reload
systemctl enable podman-auto-update.timer
systemctl start podman-auto-update.timer

===

## 1. Prerequisites
- Hetzner account
- SSH key generated locally
- Domain, if applicable

## 2. Provision the Server
- Create server in Hetzner Cloud
- Attach SSH key
- Note IPv4 / IPv6 addresses

## 3. First SSH Login
- `ssh root@<IPV4>`
- Disable password login, etc.

## 4. Install Dependencies
- Docker / Podman
- Git, etc.

## 5. Deploy the Application
- Clone repo
- Configure env vars
- Start containers / services

## 6. Updating / Redeploying
- Pull latest changes
- Restart services

## 7. Troubleshooting
- Common SSH issues
- Logs / where to look
