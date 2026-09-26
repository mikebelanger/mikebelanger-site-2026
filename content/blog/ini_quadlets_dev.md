+++
title = "A Developers Guide to Podman Quadlets"
date = "2026-09-12"
tags = ["podman", "web", "linux"]
categories = ["general"]
authors = ["mike"]
description = "A Quick Overview of Using Podman Quadlets"

[cascade.extra]
insert_anchor_links = true
+++
Here's the overall quadlet setup for a nodejs app:
```sh
├── app
│   ├── index.js
│   ├── node_modules
│   ├── package.json
│   └── package-lock.json
├── containers
│   ├── example-app.container
│   └── example-app.pod
└── update-containers.sh
```
The main benefit of using [INI-style](https://en.wikipedia.org/wiki/INI_file) Quadlets are their familiarity and clarity. System admins and UNIX greybeards alike will find the syntax familiar. The more Quadlet-specific terms can clarify where/how containers integrate:

`example-app.container`:
```ini
[Unit]
Description=Example Node.js app

[Container]
Pod=example-app.pod
Image=docker.io/node:24
WorkingDir=/app
Exec=/bin/sh -c "npm i || node --watch index.js"
Volume=../app:/app:Z

[Install]
WantedBy=multi-user.target

```

`example-app.pod`:
```ini
[Unit]
Description=Example app pod

[Pod]
PodName=example-app
ExitPolicy=stop
PublishPort=3002:3002

[Install]
WantedBy=default.target
```
##### Gotchas:
{% listicle() %}
1. ##### These aren't the actual files systemd runs
These files get translated (the technical term is "lowered") into other, *actual* systemd files for systemd to bootstrap. I like to think of the relationship quadlet files to systemd files like the relationship of SASS files to CSS, or React's JSX files to JS. While the syntax of quadlets is similar to what it gets compiled into, it isn't identical. To anyone less familiar with "pure" systemd, its difficult to distinguish between quadlet and systemd-specific declarations.

2. ##### They need to be in specific subdirectories
As someone used to `git clone`ing a repo and then doing `docker-compose up`, I found this one particularly confusing. Systemd ships with most OSs with a predefined set of paths that it looks to discover new services. Quadlets continue that trend, and have their own particular path it looks for.

Fortunately, your application code *can* live in any directory you'd like, provided that directory gets symlinked over into one of the quadlet discovery paths. A quick `ln -sfn "$PWD/containers" "$QUADLET_DIR/ini_style_quadlet"` (super intuitive, I know) should  do the trick.
{% end %}

#### Typical Dev Workflow:
{% listicle() %}
1. Developer clones code repo
2. They figure out their systemd discovery path, and symlink their downloaded repo path *into* that systemd discovery path.
3. They spin up the containers via `systemctl --user start app-name.service`
4. Add code, commit it in the usual way.
5. To stop the service, do `podman pod stop app-name.service`.

{% end %}
