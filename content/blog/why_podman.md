+++
title = "Why Podman?"
date = "2026-09-12"
tags = ["podman", "web", "linux"]
categories = ["general"]
authors = ["mike"]
description = "My experiences with using Podman"
+++
[Podman](https://podman.io) and its affiliated projects can help manage [OCI-compatible containers](https://opencontainers.org/about/overview/) in several different ways, making it more than a [Docker](https://docker.com) substitute:

{% listicle() %}
1. ###### Licensing
[Podman](https://podman.io) is Apache-2.0 licensed. Basically, it's open source and can be used by an unlimited number of users for free. If you like using [Docker Desktop](https://www.docker.com/products/docker-desktop/), its [Podman equivalent](https://podman-desktop.io/) is free and open source as well, and won't [ask you to pay if your company gets past a certain size](https://docs.docker.com/subscription-billing/desktop-license/).

2. ###### Security
Podman allows its containers to be [rootless](https://developers.redhat.com/blog/2020/09/25/rootless-containers-with-podman-the-basics#). What is rootless? It means there isn't one central process (daemon) that's in charge of all your containers. Not one central process that could crash and bring down all your containers, not one central process that could get compromised, and start acting maliciously. To be fair, Docker [has rootless mode now too](https://docs.docker.com/engine/security/rootless/), so if rootless is your only attraction, just switch to that.

3. ###### Integration with SystemD
Podman's [Quadlets](https://docs.podman.io/en/v5.8.1/markdown/podman-quadlet.1.html) facilitate integrating with [systemd](https://systemd.io/) as well. Why bother integrating with systemd? Your web app is probably running on a server with Linux, and that version of Linux is probably using systemd. systemd isn't technically part of Linux itself, but it's the unofficial standard init system for most Linux distros, including [Fedora](https://fedoraproject.org/), [Ubuntu](https://ubuntu.com/), and many others. Unless your server is running [Slackware](http://www.slackware.com/config/init.php), integrating with systemd is a no-brainer.

4. ###### Pods
As the name suggests, Podman works with sets of [pods](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/8/html/building_running_and_managing_containers/assembly_working-with-pods_building-running-and-managing-containers). Although Docker has no equivalent feature, pods aren't specific to Podman either, as [the concept was inspired from Kubernetes](https://kubernetes.io/docs/concepts/workloads/pods/). Pods allow containers to be grouped together and given their own network namespace. Once containers are grouped into “pods”, those pods can be started/restarted together, like `podman pod start my-app`.

This feature is less geared towards developers, more so system admins/devops who manage lots of containers. That said, they're easy to set up, and alleviate some of the tedium associated with spinning up/down large numbers of containers.

{% end %}

I've been using Podman for a few years now, and overall I've found it a completely acceptable replacement for Docker. That said, I've found most of the tutorials/resources (and subsequently, AI) have described Podman from a more system-admin/devops workflow, not a web developer workflow. So in the next few posts, I'm going to outline some ways I've found to work with Podman containers. Some of them will be fairly familiar (like [podman-compose](https://docs.podman.io/en/latest/markdown/podman-compose.1.html)), while others are much newer (like a [Kubernetes-style “kube” file](https://www.redhat.com/en/blog/multi-container-application-podman-quadlet)).
