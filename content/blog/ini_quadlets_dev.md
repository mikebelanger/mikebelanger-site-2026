+++
title = "Quadlets: A Developer's Perspective"
date = "2026-10-12"
tags = ["podman", "web", "linux", "oci", "devops", "selfhosting"]
categories = ["general"]
authors = ["mike"]
description = "An overview of using Podman Quadlets for web development"
[cascade.extra]
insert_anchor_links = true
+++

##### Introduction

[Podman's](https://podman.io/) [Quadlet feature](https://docs.podman.io/en/stable/markdown/podman-quadlet.1.html) allows podman containers to be integrated into [systemd](https://systemd.io/) services. 

Why manage containers with systemd? You might argue that containers are better orchestrated with [Docker Swarm](https://docs.docker.com/engine/swarm/), [Kubernetes](https://kubernetes.io/), or even [Ansible](https://www.ansible.com/).  That is true for larger installations. For a single Linux box or VPS, systemd is probably running on it already. I couldn't summarize it better than Elias Bourgess: [it's the control pane you already run](https://ebourgess.dev/posts/podman-quadlet-production/#the-control-plane-you-already-run).

There are two ways to write Quadlets:
- [INI-style quadlet definitions.](https://www.redhat.com/en/blog/quadlet-podman.) An extension of systemd's INI-style config language. This adds sections for things [like containers and volumes](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html). This approach is for Linux system administrators who are already familiar with systemd.
- [K8s-style quadlet definitions.](https://www.redhat.com/en/blog/compose-podman-pods) These are Kubernetes-style yaml files that define your containers. This approach is clearly aimed at developers and devops folks already familiar with [Kubernetes](https://kubernetes.io/) or [docker-compose](https://docs.docker.com/compose/).

{% summary(summary = "Caveats") %}
* You may be wondering why [podman-compose](https://github.com/containers/podman-compose) isn't mentioned as an option.  While podman-compose is obviously a docker-compose replacement, industry leaders are shying away from supporting it, in favor of `podman kube play`. More on that [here.](https://www.redhat.com/en/blog/podman-compose-docker-compose) 

* In order to get *k8*-style Kubernetes yaml, you need a [*kube file*](https://docs.podman.io/en/latest/markdown/podman-kube.unit.5.html), which is technically an INI-style file. I'm glossing over that for now, as the bulk of the container definitions are still either expressed in Kubernetes-style YAML or INI.
{% end %}

I'm a developer who's used to docker-compose, so I [tried out the k8-style first](https://github.com/mikebelanger/lucky-vanilla/blob/main/containers/prod/pod.yml). That said, curiosity got the better of me, and I wondered what INI-style Quadlet definitions would look like.

##### Setup
----

Here's a minimal setup for a Node.js application (full source code [can be found here](https://github.com/mikebelanger/ini_style_quadlet)):
```sh
├── app
│   ├── index.js
│   ├── node_modules
│   ├── package.json
│   └── package-lock.json
├── dev
│   ├── example-app-dev.container
│   └── example-app-dev.pod
├── prod
│   ├── Containerfile
│   ├── example-app.build
│   ├── example-app.container
│   └── example-app.pod
└── README.md
```

The quadlet-specific files here are under the `dev` and `prod` subdirectories for development and production, respectively. Here's a look at the ones in `dev`:

`dev/example-app-dev.container`:
```ini
[Unit]
Description=Example Node.js app

[Container]
Pod=example-app-dev.pod
Image=docker.io/node:24
WorkingDir=/app
Exec=/bin/sh -c "npm i && node --watch index.js"
Volume=../app:/app:Z
Environment=PORT=3001

[Install]
WantedBy=multi-user.target
```

`dev/example-app-dev.pod`:
```ini
[Unit]
Description=Development version of example app

[Pod]
PodName=example-app-dev
ExitPolicy=stop
PublishPort=3001:3001

[Install]
WantedBy=default.target
```
{% summary(summary = "Why wrap one container in a Pod?")%}
You may be wondering why it's wrapped in a Pod. For this example, the Pod is indeed overkill. Once other containers get added to your app (a Postgres db container, a Redis cache layer, reverse-proxy, etc) pods do make it easier to manage containers, and keep them all talking to each other with relative ease. This is because containers on the same pod can all access each other via `localhost:PORT`, and not worry about whatever host addresses are already in use. Furthermore, at the pod level, only the port that needs access from the outside world (say, a reverse proxy's container port) is published, while the other containers' ports are effectively closed off. 
{% end %}

{% summary(summary = "Notes") %}
1. ###### These aren't the actual files systemd runs
These files get translated (the technical term is "lowered") into other, *actual* systemd files for systemd to bootstrap. I like to think of the relationship quadlet files to systemd files like the relationship of SASS files to CSS, or React's JSX files to JS. While the syntax of quadlets is similar to what it gets compiled into, it isn't identical. To anyone less familiar with "pure" systemd, it's difficult to distinguish between quadlet and systemd-specific declarations.

2. ###### Systemd needs application code in certain subdirectories in order to run it
I'm used to cloning repos onto any path on my disk, doing `docker-compose up` in that repo, and I'm up and running. Unfortunately, the Quadlet workflow isn't quite as straightforward.  Systemd ships with most OSs with a predefined set of paths that it looks to discover new services. Quadlets continue that trend.

Fortunately, your application code *can* live in any directory you'd like, provided that directory gets copied/symlinked over into one of the quadlet discovery paths. A quick `ln -sfn "$PWD" ~/.config/containers/systemd/example-app` (super intuitive, I know) should do the trick.

3. ###### Mind the volume mount
`Volume=../app:/app:Z` mounts your source directory read/write, and the `:Z` suffix relabels that directory for this one container's private use. That's the right choice for a single container. But if a *second* container in a different pod ever mounts the same directory, it gets its own SELinux label and one of the two will fail with a "permission denied". If you genuinely intend to share a directory, use `:z` (shared) instead.

One more thing worth knowing: the above assumes *rootless* podman (`systemctl --user`), where the container's `root` maps to your own user, so anything it writes to `../app` stays yours to edit and delete. Run the same directory through `sudo podman` even once, though, and the files it creates become owned by real root, leaving you reaching for `sudo` to clean them up.
{% end %}

#### Dev Workflow:
1. Clone the repo.
1. Find your systemd discovery path. On most systems, that resolves to `${XDG_CONFIG_HOME:-$HOME/.config}/containers/systemd`. So something like `~/.config/containers/systemd`. (If you're doing rootless, which I am)
1. `cd` into your repo, and link up the repo's subdirs into your quadlet discovery path:
```sh
cd project-dir

# symlink the entire project directory over. Note that "example-app" can be anything, and should be uniquely named
# to prevent clashes with other systemd services
ln -sfn "$PWD" ~/.config/containers/systemd/example-app
```
Now reload systemd with this new application, and confirm all is well
```sh
# This rebuilds the quadlets -> actual systemd runtime files
systemctl --user daemon-reload

# Confirm that quadlet picked it up with `podman quadlet list`. If all went well, you'll see your pod listed.
podman quadlet list

# Will probably give you something like
# NAME                       UNIT NAME                    PATH ON DISK                                       STATUS          APPLICATION
# example-app-dev.container  example-app-dev.service      /your/dev-dir/dev/example-app-dev.container        active/running  .dev.app
# example-app-dev.pod        example-app-dev-pod.service  /your/dev-dir/dev/example-app-dev.pod              active/running  .dev.app

# Check the status of the service
systemctl --user status example-app-dev.service

# BONUS: If you'd prefer your services to keep running after you log out (and start at boot), enable lingering for your user
sudo loginctl enable-linger $USER
```
### Now to the actual developing!
The main source files under `app` are volume-mounted, so go ahead, start changing `index.js`, for example. Reload the browser to see your changes.

#### Deployment Workflow

If you're deploying to the same system you develop on, then all you're really doing is `COPY`ing the node source files instead of volume-mounting. If you're using Typescript or anything else requiring compilation, you'll have to add that to your `Containerfile` too.

`prod/example-app.container`:
```ini
[Unit]
Description=Example Node.js app

[Container]
Pod=example-app.pod
Image=example-app.build
Exec=/bin/sh -c "node index.js"
Environment=PORT=3005

[Install]
WantedBy=default.target
```

`prod/example-app.pod`:
```ini
[Unit]
Description=Production version of example app

[Pod]
PodName=example-app-pod
ExitPolicy=stop
PublishPort=3005:3005

[Install]
WantedBy=default.target
```
`prod/Containerfile`:
```Dockerfile
FROM docker.io/library/node:24-alpine

WORKDIR /app

COPY app/ /app/

RUN npm ci
```

`prod/example-app.build`:
```ini
[Build]
ImageTag=localhost/example-app
File=Containerfile
SetWorkingDirectory=..
```
You've already linked the production files over, its just a matter of rebuilding.

```sh
# Rebuilds any necessary files
systemctl --user daemon-reload

# Start the production service
systemctl --user restart example-app.service
```
Point a browser at `localhost:3005` and you should see your production app!

{% summary(summary = "What about Quadlet Install?")%}
The short answer is, eventually I think we should. I'm only on *Podman 5.8.7* (for the latest stable Fedora, as of this writing). On this version, `podman quadlet install` effectively copies over all your source code files into [*the base quadlet discovery directory*](https://github.com/podman-container-tools/podman/blob/4c3027c149ad54dbb6f96328694e7b852623b4b8/pkg/domain/infra/abi/quadlet.go#L147-L162). If you're managing multiple services/quadlets, this would fill up full of `.container`'s and `.pods` pretty fast! To solve this, they add an `.app` file to denote which `.container` files belong to which, but this seems incredibly messy to me. Fortunately, they've changed this in Podman 6, and allow services to be grouped under different directory names [(via the `--application` flag)](https://docs.podman.io/en/stable/markdown/podman-quadlet-install.1.html#application-string).

In other words, once Fedora upgrades, I'd go with something like `podman quadlet install --application="example-app-dev" ..` instead.

{% end %}

#### Final Thoughts

Honestly, I prefer the k8-style workflow over this. The idea of linking my project subdirectory into an arbitrary system path to start my service just feels so, *alien*. Even with the changes to Podman 6 and the `--application` flag, it feels like an odd workflow. 

On the flipside, INI-style quadlets *feel* "closer to the metal". Having everything already written in systemd INI files is a pretty close representation of what actually gets loaded by systemd. There's a movement in the frontend ecosystem which eschews elaborate frameworks for plain "vanilla" javascript, and to [just use the platform](https://elisehe.in/2021/08/22/using-the-platform). I felt like there was a parallel to be found with Quadlets, and that was worth exploring.
