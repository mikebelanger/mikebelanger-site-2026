+++
title = "INI-style quadlets: a developer's perspective"
date = "2026-09-27"
tags = ["podman", "web", "linux"]
categories = ["general"]
authors = ["mike"]
description = "A Quick Overview of Using Podman Quadlets For Web Development"
draft = true
[cascade.extra]
insert_anchor_links = true
+++

##### Introduction

[Podman's](https://podman.io/) [Quadlet feature](https://docs.podman.io/en/stable/markdown/podman-quadlet.1.html) allow podman containers to be integrated into [systemd](https://systemd.io/) services. 

Why manage containers with systemd? You might argue that containers are better orchestrated with [Docker Swarm](https://docs.docker.com/engine/swarm/), [Kubernetes](https://kubernetes.io/), or even [Ansible](https://www.ansible.com/).  That is true for larger installations. For a single Linux box or VPS, systemd is probably running on it already. I couldn't summarize it better than Elias Bourgess: [it's the control pane you already run.](https://ebourgess.dev/posts/podman-quadlet-production/#the-control-plane-you-already-run).

There's two ways to write Quadlets:
- [INI-style quadlet definitions.](https://www.redhat.com/en/blog/quadlet-podman.) An extension of systemd's INI-style config language. This adds sections for things [like containers and volumes](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html). This approach is for Linux system administrators who are already familiar with systemd.
- [K8s-style quadlet definitions.](https://www.redhat.com/en/blog/compose-podman-pods) These are Kubernetes-style yaml files that define your containers. This approach is clearly aimed at more dev/devops people. Anyone familiar with [Kubernetes](https://kubernetes.io/) or [docker-compose](https://docs.docker.com/compose/).

{% summary(summary = "CAVEATS") %}
* You may be wondering why [podman-compose](https://github.com/containers/podman-compose) isn't mentioned as an option.  While podman-compose is obviously a docker-compose replacement, industry leaders are shying away from supporting it, in favor `podman kube play`. More on that [here.](https://www.redhat.com/en/blog/podman-compose-docker-compose) 

* In order to get *k8*-style kubernetes yaml, you need a [*kube file*](https://docs.podman.io/en/latest/markdown/podman-kube.unit.5.html), which is technically an INI-style file. I'm glossing over that for now, as the bulk of the container definitions are still either expressed in Kubernetes-style YAML or INI.
{% end %}

I'm a developer who's used to docker-compose, so I [tried out the k8-style first](https://github.com/mikebelanger/lucky-vanilla/blob/main/containers/prod/pod.yml). That said, curiosity got the better of me, and wondered what INI-style Quadlet definitions would look like.

##### Setup
----

Here's a minimal setup for a nodejs application:
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
│   ├── example-app.container
│   └── example-app.pod
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

[Install]
WantedBy=multi-user.target
```

`dev/example-app-dev.pod`:
```ini
[Unit]
Description=Development version of example app

[Pod]
PodName=example-app-dev-pod
ExitPolicy=stop
PublishPort=3002:3002

[Install]
WantedBy=default.target
```

{% summary(summary = "NOTES") %}
##### Important Concepts:
1. ###### These aren't the actual files systemd runs
These files get translated (the technical term is "lowered") into other, *actual* systemd files for systemd to bootstrap. I like to think of the relationship quadlet files to systemd files like the relationship of SASS files to CSS, or React's JSX files to JS. While the syntax of quadlets is similar to what it gets compiled into, it isn't identical. To anyone less familiar with "pure" systemd, its difficult to distinguish between quadlet and systemd-specific declarations.

2. ###### Systemd needs application code in certain subdirectories in order to run it
I'm used to cloning repos onto any path on my disk, doing `docker-compose up` in that repo, and I'm up and running. Unfortunately, the Quadlet workflow isn't quite as straightforward.  Systemd ships with most OSs with a predefined set of paths that it looks to discover new services. Quadlets continue that trend.

Fortunately, your application code *can* live in any directory you'd like, provided that directory gets copied/symlinked over into one of the quadlet discovery paths. A quick `ln -sfn "$PWD/containers" "$QUADLET_DIR/ini_style_quadlet"` (super intuitive, I know) should  do the trick.
{% end %}

#### Dev Workflow:
1. Clone the repo.
1. Find your systemd discovery path. On most systems, that resolves to `${XDG_CONFIG_HOME:-$HOME/.config}/containers/systemd`. So something like `~/.config/containers/systemd`. (If you're doing rootless, which I am)
1. `cd` into your repo, and link up the repo's subdirs into your quadlet discovery path:
```sh
cd project-dir
ln -sfn "$PWD/dev" ~/.config/containers/systemd/dev
ln -sfn "$PWD/app" ~/.config/containers/systemd/app
```
{% summary(summary = "WHAT?") %}
Yeah, I'm not a fan of sym-linking over like this either.

Some of you might ask why not just use [`podman quadlet install`](https://docs.podman.io/en/stable/markdown/podman-quadlet-install.1.html#name), which is designed to make an "easy installation". In terms of development, `podman quadlet install` copies over files into the discovery directory. If you're looking at saving volume-mounted source code files, you'd have to repeatedly do `podman quadlet install` for any new updates. So development-wise, `ln` is necessary. That, or simply do your application development straight in your quadlet discovery directory.
{% end %}

Now reload systemd with this new application, and confirm all is well
```sh
# Reload the system with
systemctl --user daemon-reload

# Confirm that quadlet picked it up with `podman quadlet list`. If all went well, you'll see your pod listed.
podman quadlet list

# Start the service
systemctl --user restart example-app-dev-pod.service

# Check the status of the service
systemctl --user status example-app-dev.service

# BONUS: If you'd prefer this always be on when logging in, ensure loginctl enables this to start on every new login
sudo loginctl enable-linger $USER
```

### Now to the actual developing!
The main sources files under `app` are volume-mounted, so go ahead, start changing `main.js`, for example. Reload the browser to see the effects change.

#### Deployment
1. Pretty much the same as above, except `cp` instead of link

```sh
# Copy application code and production container definitions
cp -rf app ~/.config/containers/systemd/.
cp -rf prod ~/.config/containers/systemd/.
```

Fill this out
{% summary(summary = "WHAT ABOUT QUADLET INSTALL?")%}
The short answer is, eventually I think we should. I'm only on *Podman 5.8.7* (for the latest stable Fedora, as of this writing). On this version, `podman quadlet install` effectively copies over all your source code files into [*the base quadlet discovery directory*](https://github.com/podman-container-tools/podman/blob/4c3027c149ad54dbb6f96328694e7b852623b4b8/pkg/domain/infra/abi/quadlet.go#L147-L162). If you're managing multiple services/quadlets, this would fill up full of `.container`'s and `.pods` pretty fast! To solve this, they add an `.app` file to denote which `.container` files belong to which, but this seems incredibly messy to me. Fortunately, they've changed this in Podman 6, and allow services to be grouped under different directory names [(via the `--application` flag)](https://docs.podman.io/en/stable/markdown/podman-quadlet-install.1.html#application-string).

In other words, once Fedora upgrades, I'd go with something like `podman quadlet install --application="example-app-dev" ..` instead.

{% end %}

#### Final Thoughts

Honestly, I prefer the k8-style workflow over this. The idea of linking my project subdirectory into an arbitrary system path to start my service just feels so, *alien*. Even with the changes to Podman 6 and the `--application` flag, it feels like an odd workflow. One thing the INI-style quadlet workflow has is it *feels* "closer to the metal". Having everything already written in a systemd-INI files is a pretty close representation of what actually gets loaded by systemd. Whereas with `podman kube play`, I making some guesses as to what those units look like.
