+++
title = "A Developer's Guide to Podman"
date = "2026-09-12"
tags = ["podman", "web", "linux"]
categories = ["general"]
authors = ["mike"]
description = "An opinionated guide to web dev with Podman"

[cascade.extra]
insert_anchor_links = true
+++
{% summary(summary="Summary")  %}
[Podman](https://podman.io) provides a number of ways for developers to manage containers:
- [Makefiles and shell scripts](#makefiles-and-shell-scripts)
- [Podman-compose](#podman-compose)
- [INI-style quadlets](#quadlets-ini-style)
- [Kubernetes-style quadlets](#quadlets-kubernetes-kube-style)
- [Ansible](#ansible)
{% end %}

### Introduction
---
[Podman](https://podman.io) and its affiliated projects can help manage [OCI-compatible containers](https://opencontainers.org/about/overview/) in several different ways, making it more than a [docker](https://docker.com) substitute:

{% listicle() %}
1. ###### Licensing
[Podman](https://podman.io) is Apache-2.0 licensed (basically open-source, free-to-use). If you like using [docker-desktop](https://www.docker.com/products/docker-desktop/), its [Podman equivalent](https://podman-desktop.io/) is open-source as well, and won't [ask you to pay if your company gets past a certain size](https://docs.docker.com/subscription-billing/desktop-license/).

2. ###### Security
Podman allows its containers to be [rootless](https://developers.redhat.com/blog/2020/09/25/rootless-containers-with-podman-the-basics#). What is rootless? It means there isn't one central process (daemon) that's in charge of all your containers. Not one central process that could crash and bring down all your containers, not one central process that could get compromised, and start acting maliciously. To be fair, docker does now have rootless mode, so if rootless is your only attraction, just switch to that. 

3. ###### Deeper integration with systemd
Your web app is probably running on a server with Linux, and that version of Linux is probably using [systemd](https://systemd.io/). Systemd isn't technically required in Linux, but its the unofficial standard task scheduler for most Linux distros, including [Fedora](https://fedoraproject.org/), [Ubuntu](https://ubuntu.com/), [Arch](https://archlinux.org/), and many others. Unless your server is running [Slackware](http://www.slackware.com/config/init.php), integrating with systemd is a no-brainer.

Some of you may already be used to writing simple systemd configurations to bootstrap a docker process, and might be wondering how Podman goes beyond this. The answer to this is [Quadlets](https://www.redhat.com/en/blog/quadlet-podman), which this post will expand on later.

4. ###### Pods
As its name implies, Podman works with sets of [pods](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/8/html/building_running_and_managing_containers/assembly_working-with-pods_building-running-and-managing-containers). Although Docker has no equivalent feature, pods aren't specific to Podman either, as [it took the concept from Kubernetes](https://kubernetes.io/docs/concepts/workloads/pods/). Pods allow containers to be grouped together and given their own network namespace. Once containers are grouped into "pods", those pods can be started/restarted together, like `podman pod start my-app`.

This feature is less geared towards developers, more-so system admins/devops who manage lots of containers. That said, they're easy to setup, and alleviate some of tedium associated with spinning up/down large swaths of containers.

{% end %}

### Different Approaches to Managing Podman Containers
---

I've seen Podman used in a variety of ways, and I'd classify those ways into these categories:

1. #### Makefiles and shell-scripts
2. #### Podman-Compose
3. #### Quadlets (INI-style)
4. #### Quadlets (Kubernetes-kube style)
5. #### Ansible

The typical way of declaring quadlets is through a series of [INI-style](https://en.wikipedia.org/wiki/INI_file) files, more or less the same as [systemd](https://systemd.io/):

`app-db.container`:
```ini
[Unit]
Description=PostgreSQL for development

[Container]
Image=docker.io/library/postgres:16
ContainerName=app-db
PublishPort=5432:5432
Volume=app-db-data.volume:/var/lib/postgresql/data
Environment=POSTGRES_PASSWORD=dev

[Service]
Restart=on-failure

[Install]
WantedBy=default.target
```

Separately, a volume is declared in its own quadlet file:

`app-db-data.volume`:
```ini
[Volume]
VolumeName=app-db-data
Driver=local
```

And a simple application layer ties the pieces together:

`app.container`:
```ini
[Unit]
Description=My web application
Requires=app-db.container
After=app-db.container

[Container]
Image=localhost/app:latest
ContainerName=app
PublishPort=8000:8000
Environment=DATABASE_URL=postgres://postgres:dev@app-db:5432/dev

[Service]
Restart=always

[Install]
WantedBy=default.target
```

As you can see, unlike docker-compose, everything is broken into different files.

## Sharing a single `.env` file

That split keeps each unit independently managed by systemd, but the same values — the database password, the port — end up hard-coded in several places. A shared `.env` file cuts down on that repetition:

`.env`:
```ini
# Database
POSTGRES_USER=postgres
POSTGRES_PASSWORD=dev
DB_NAME=development
DB_PORT=5432

# Application
HOST=0.0.0.0
PORT=8000
```

Keep this file next to your quadlets, under `~/.config/containers/systemd/`. Load it in each unit's `[Service]` section and reference the variables with `${VAR}`:

`app-db.container`:
```ini
[Unit]
Description=PostgreSQL for development

[Container]
Image=docker.io/library/postgres:16
ContainerName=app-db
PublishPort=${DB_PORT}:${DB_PORT}
Volume=app-db-data.volume:/var/lib/postgresql/data
Environment=POSTGRES_PASSWORD=${POSTGRES_PASSWORD}

[Service]
EnvironmentFile=%h/.config/containers/systemd/.env
Restart=on-failure

[Install]
WantedBy=default.target
```

`app.container`:
```ini
[Unit]
Description=My web application
Requires=app-db.container
After=app-db.container

[Container]
Image=localhost/app:latest
ContainerName=app
PublishPort=${PORT}:${PORT}
Environment=HOST=${HOST}
Environment=PORT=${PORT}
Environment=DATABASE_URL=postgres://postgres:dev@app-db:${DB_PORT}/dev

[Service]
EnvironmentFile=%h/.config/containers/systemd/.env
Restart=always

[Install]
WantedBy=default.target
```

Quadlet passes the `${VAR}` references straight into the systemd unit it generates — the `ExecStart` ends up looking like `podman run ... --publish ${PORT}:${PORT} --env DATABASE_URL=postgres://postgres:dev@app-db:${DB_PORT}/dev` — and systemd expands them at launch time, using the environment loaded from the file. Define each value once in `.env`, reference it in every file, and a single edit reconfigures the whole stack. Note that `DATABASE_URL` is composed in the quadlet file from `${DB_PORT}`, not stored as a whole in `.env`.

A few details worth knowing:

- There are two different keys that look alike. `[Service] EnvironmentFile=` feeds the host-side environment that systemd expands `${VAR}` from — that's what makes `${PORT}` work inside `PublishPort=` and every other line. `[Container] EnvironmentFile=` is a separate, equally valid key that maps to `podman run --env-file`: it loads the file *into* the container and the host never sees those variables. If the `.env` values are only needed inside the container, `[Container] EnvironmentFile=` alone is enough and you can drop the `Environment=` lines — but then nothing host-side (like the `PublishPort=` line) can expand them.
- `.volume` and `.network` units don't support `Environment` or `EnvironmentFile` keys at all, so keep shared variables in your `.container` files (pods load them through their `[Service]` section, like the containers do).
- `${VAR}` interpolation only happens on the generated `ExecStart`/`ExecStartPre` lines — not inside the `.env` file itself. `EnvironmentFile` values are literal, so referencing another variable inside a `.env` value (like writing `${DB_PORT}` into a `DATABASE_URL` line there) would hand the literal string to the container. Compose such values in the quadlet file instead, as above.
- `%h` expands to your home directory, which fits the rootless layout. If you're using system quadlets under `/etc/containers/systemd/`, point `EnvironmentFile` at the absolute path instead.
- The file must exist when the unit starts. The `-` prefix that makes other systemd environment files optional isn't handled yet ([containers/podman#26719](https://github.com/podman-container-tools/podman/issues/26719)).
- Avoid `%N`-style specifiers in resource names like `Volume=app-db-data.volume`: specifiers resolve per generated unit, so they point at the generated unit's own name instead of the resource you meant, and Quadlet can't link the files. `${VAR}` references don't have this problem.

## Wrapping the whole stack in a pod

Those three files can also be attached to a single pod. Containers in a pod share a network namespace (and cgroups), so they talk to each other over `localhost` instead of discrete hostnames.

The pod owns that shared network namespace, so anything with network scope — published ports, the network itself, aliases, DNS, hostname — is declared in the pod's own file, and nowhere else. Containers keep only process-scoped configuration: the image, environment, volumes. The `.pod` file doubles as the declaration of the pod's exposed surface, which is exactly what it reads like:

`app.pod`:
```ini
[Unit]
Description=My application pod

[Pod]
PodName=app-pod
PublishPort=${PORT}:${PORT}

[Service]
EnvironmentFile=%h/.config/containers/systemd/.env

[Install]
WantedBy=default.target
```

The pod needs the same `EnvironmentFile` under `[Service]` as the containers — `PublishPort` ends up on the pod's `ExecStartPre` line, and that's what gives systemd the `PORT` value to expand.

Each container joins the pod with a single `Pod=` line, and port publishing moves up to the pod — the containers drop their own `PublishPort`:

`app-db.container`:
```ini
[Unit]
Description=PostgreSQL for development

[Container]
Image=docker.io/library/postgres:16
ContainerName=app-db
Pod=app.pod
Volume=app-db-data.volume:/var/lib/postgresql/data
Environment=POSTGRES_PASSWORD=${POSTGRES_PASSWORD}

[Service]
EnvironmentFile=%h/.config/containers/systemd/.env
Restart=on-failure

[Install]
WantedBy=default.target
```

`app.container`:
```ini
[Unit]
Description=My web application

[Container]
Image=localhost/app:latest
ContainerName=app
Pod=app.pod
Environment=HOST=${HOST}
Environment=PORT=${PORT}
Environment=DATABASE_URL=postgres://postgres:dev@localhost:${DB_PORT}/dev

[Service]
EnvironmentFile=%h/.config/containers/systemd/.env
Restart=always

[Install]
WantedBy=default.target
```

How this plays out:

- `app.pod` becomes `app-pod.service`. Quadlet wires the stack together automatically: the pod unit gets `Wants=`/`Before=` for each joined container, and each container unit gets `BindsTo=`/`After=` on the pod. `systemctl --user start app-pod.service` brings up everything; stopping the pod stops its containers.
- Containers inside the pod share its network, so the app reaches Postgres at `localhost` — that's why the `DATABASE_URL` host switches from `@app-db` to `@localhost`. The host portion is written inline because it depends on the topology, but the port still comes from `.env` via `${DB_PORT}`.
- Only the pod publishes ports now. Postgres' internal `5432` is no longer reachable from the host at all, which is a nice side effect — the database is only reachable from inside the pod.
- Podman backs up that division: a container joined to a pod cannot declare its own network config. `PublishPort=`, `Network=`, or `podman run --publish` on a pod member is rejected before the container even starts — `network cannot be configured when it is shared with a pod`. So the port can only live in `app.pod`, and the container files can never drift out of sync with it.
- Pods default to `ExitPolicy=stop`: when the last container in the pod exits, the pod itself stops. If you want the pod to stick around for tooling, set `ExitPolicy=continue` under `[Pod]`.
- `podman kube generate app-pod` emits the equivalent single-Pod YAML, which is where the "Kube way" in the title comes in — more on that in a later post.

## Keeping quadlets in your repository

So far everything lives in `~/.config/containers/systemd/` (the default search path for rootless user quadlets). Putting the files directly under your home directory works, but they're plain text — they belong in version control. Quadlet follows symbolic links inside its search paths, including a symlinked subdirectory, so a single symlink points the whole search path at your repository:

`ln -s ~/code/app/containers/systemd ~/.config/containers/systemd/app`

Now `app.pod`, `app-db.container`, `app-db-data.volume` and friends live in the repo and are discovered through the `app` symlink every time units are generated. Since Quadlet regenerates the units on `systemctl --user daemon-reload`, the whole workflow is: edit files in the repo, commit, pull on the target machine, `daemon-reload`, done. There's no separate install step to keep in sync with the sources.

A few notes:

- The `.env` file doesn't have a quadlet extension, so it isn't scanned — but `EnvironmentFile` reads it at runtime all the same. Point it through the symlink too and it stays in the repo: `EnvironmentFile=%h/.config/containers/systemd/app/.env` (keep the symlink name consistent across machines, since it's part of the path).
- The generator resolves through the symlink, so error messages and `SourcePath` reference the real repo files, not the link.
- If you'd rather not keep a live symlink, `podman quadlet install <file>` (Podman 5.6+) copies a unit file into the search path for you. Running Quadlet straight from an app subdirectory without any symlink is an open feature request ([containers/podman#26941](https://github.com/podman-container-tools/podman/issues/26941), [containers/podman#27069](https://github.com/podman-container-tools/podman/issues/27069)).
