# Exercises – Containers: Docker and Podman

<!-- nav:start -->
> **Module 08 · Security, Virtualization & Containers** — lesson 6 of 6  
> Read first: [Theory – Containers: Docker and Podman](../../theory/08-security-vms-containers/06-containers.md) · [Course lesson (commands)](../../course/08-Security-VMs-Containers.md#lesson-6---containers-ch-18--gap) · [Module exercises](README.md)
<!-- nav:end -->

> Podman on `rhel1`, Docker on `deb1`. Commands are interchangeable (`docker` ↔ `podman`) unless noted.

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Exercise 1 – Install and first run ★

Install the engine on each VM, show engine info (storage location, rootless?), run `alpine echo hello` once with auto-removal.

<details>
<summary>Solution</summary>

```bash
dnf install -y podman; podman info | grep -E 'graphRoot|rootless'
apt install -y docker.io; systemctl enable --now docker; docker info | grep -E 'Docker Root Dir|Storage Driver'
podman run --rm docker.io/library/alpine echo hello
```

</details>

---

## Exercise 2 – Images ★

Pull `nginx:alpine` and `redis:alpine`; list images; show the layers and the default command of nginx; remove redis.

<details>
<summary>Solution</summary>

```bash
podman pull docker.io/library/nginx:alpine docker.io/library/redis:alpine
podman images; podman history nginx:alpine
podman image inspect nginx:alpine --format '{{.Config.Cmd}}'
podman rmi redis:alpine
```

</details>

---

## Exercise 3 – Run and manage (exam-style) ★★

> Run `nginx:alpine` as a container named `web`, publishing port 8080, serving `/srv/site` (bind mount, read-only), with restart policy `always`.

Then: show logs, run `nginx -t` inside it, show its IP and restart policy, stop/start it.

<details>
<summary>Solution</summary>

```bash
mkdir -p /srv/site && echo "<h1>container</h1>" > /srv/site/index.html
podman run -d --name web -p 8080:80 -v /srv/site:/usr/share/nginx/html:ro,Z --restart always docker.io/library/nginx:alpine
curl -s localhost:8080
podman logs web; podman exec web nginx -t
podman inspect web --format '{{.NetworkSettings.IPAddress}} {{.HostConfig.RestartPolicy.Name}}'
podman stop web; podman start web
```

</details>

---

## Exercise 4 – Without :Z ★★

On `rhel1`, recreate `web` **without** `:Z`. What happens when you request the page? Find the evidence and explain.

<details>
<summary>Solution</summary>

403/permission denied in `podman logs web`; `ausearch -m AVC -ts recent` shows `container_t` denied reading `var_t` files. `:Z` relabels the directory to `container_file_t`.

</details>

---

## Exercise 5 – Volumes persist ★★

> Run container `cache` from `redis:alpine`, restart policy always, port 6379 published, data volume `redisdata` at `/data`.

Write a key, remove the container, recreate it with the same volume, read the key.

<details>
<summary>Solution</summary>

```bash
podman run -d --name cache --restart always -p 6379:6379 -v redisdata:/data docker.io/library/redis:alpine
podman exec cache redis-cli set greeting hello
podman exec cache redis-cli save
podman rm -f cache
podman run -d --name cache --restart always -p 6379:6379 -v redisdata:/data docker.io/library/redis:alpine
podman exec cache redis-cli get greeting
podman volume inspect redisdata --format '{{.Mountpoint}}'
```

</details>

---

## Exercise 6 – Networks ★★

Create network `appnet`. Run `redis:alpine` as `db` and an `alpine` container `client` on it; from `client`, reach `db` by name.

<details>
<summary>Solution</summary>

```bash
podman network create appnet
podman run -d --name db --network appnet docker.io/library/redis:alpine
podman run --rm --network appnet docker.io/library/alpine sh -c 'getent hosts db && nc -zv db 6379'
```

</details>

---

## Exercise 7 – Build an image ★★

Write a Containerfile based on `nginx:alpine` that copies your own `index.html`, installs `curl`, and has a healthcheck. Build it as `mysite:1.0`, run it on port 8082, check `podman healthcheck run` (Podman) or `docker ps` health status.

<details>
<summary>Solution</summary>

```bash
mkdir -p ~/mysite && cd ~/mysite && echo "<h1>mysite 1.0</h1>" > index.html
cat > Containerfile << 'EOF'
FROM docker.io/library/nginx:alpine
COPY index.html /usr/share/nginx/html/
RUN apk add --no-cache curl
HEALTHCHECK CMD curl -fs http://localhost/ || exit 1
EOF
podman build -t mysite:1.0 .
podman run -d --name mysite -p 8082:80 mysite:1.0
curl -s localhost:8082; podman healthcheck run mysite && echo healthy
```

</details>

---

## Exercise 8 – Limits and inspection ★

Run a container limited to 128 MiB of memory and 0.5 CPU. Check with `stats`. Try to allocate more memory inside it and see what happens.

<details>
<summary>Solution</summary>

```bash
podman run -d --name limited --memory 128m --cpus 0.5 docker.io/library/alpine sleep 3600
podman stats --no-stream limited
podman exec limited sh -c 'head -c 300m /dev/zero | tail' ; echo "exit=$?"    # OOM-killed (137)
```

</details>

---

## Exercise 9 – Start at boot with Quadlet ★★★

On `rhel1`, remove the `web` container and recreate it as a Quadlet unit so systemd starts it at boot. Reboot and verify.

<details>
<summary>Solution</summary>

```bash
podman rm -f web
cat > /etc/containers/systemd/web.container << 'EOF'
[Unit]
Description=Web container
[Container]
Image=docker.io/library/nginx:alpine
ContainerName=web
PublishPort=8080:80
Volume=/srv/site:/usr/share/nginx/html:ro,Z
[Service]
Restart=always
[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload && systemctl start web.service
systemctl status web.service; reboot; podman ps
```

</details>

---

## Exercise 10 – Engine configuration ★★

1. Docker: limit container logs to 10 MB × 3 files via `daemon.json`; restart Docker and confirm with `docker info`.
2. Podman: add a search-registry drop-in so that `podman pull nginx` resolves to `docker.io` without asking.

<details>
<summary>Solution</summary>

```bash
cat > /etc/docker/daemon.json << 'EOF'
{ "log-driver": "json-file", "log-opts": { "max-size": "10m", "max-file": "3" } }
EOF
systemctl restart docker; docker info | grep -i 'logging driver'
echo 'unqualified-search-registries = ["docker.io"]' > /etc/containers/registries.conf.d/99-lab.conf
podman pull nginx:alpine
```

</details>

---

## Challenge – Rootless service ★★★

As user `ana` (no sudo), run the `mysite:1.0` image as a **rootless** Quadlet service on port 8083 that keeps running after `ana` logs out and starts at boot.

<details>
<summary>Solution</summary>

```bash
loginctl enable-linger ana            # as root, once
su - ana
mkdir -p ~/.config/containers/systemd
podman build -t mysite:1.0 ~/mysite   # build it as ana (rootless storage)
cat > ~/.config/containers/systemd/mysite.container << 'EOF'
[Container]
Image=localhost/mysite:1.0
PublishPort=8083:80
[Install]
WantedBy=default.target
EOF
systemctl --user daemon-reload && systemctl --user start mysite.service
curl localhost:8083
```

(Use `XDG_RUNTIME_DIR=/run/user/$(id -u)` if `systemctl --user` complains after `su`; logging in via SSH as ana avoids this.)

</details>

---

## Cleanup

```bash
podman rm -af; podman volume prune -f; podman network rm appnet
rm -f /etc/containers/systemd/web.container; systemctl daemon-reload
```

---

<!-- footer:start -->
[← Exercises: Virtual Machines with libvirt](05-libvirt-vms.md) · [Module exercises](README.md)
<!-- footer:end -->
