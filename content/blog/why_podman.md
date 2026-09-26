+++
title = "Why Podman?"
date = "2026-09-12"
tags = ["podman", "web", "linux"]
categories = ["general"]
authors = ["mike"]
description = "My experiences with using Podman"

[cascade.extra]
insert_anchor_links = true
+++
[Podman](https://podman.io) and its affiliated projects can help manage [OCI-compatible containers](https://opencontainers.org/about/overview/) in several different ways, making it more than a [docker](https://docker.com) substitute:

{% listicle() %}
1. ###### Licensing
[Podman](https://podman.io) is Apache-2.0 licensed. Basically, its open-source, and can be used by an unlimited number of users for free. If you like using [docker-desktop](https://www.docker.com/products/docker-desktop/), it's [Podman equivalent](https://podman-desktop.io/) is free and open-source as well, and won't [ask you to pay if your company gets past a certain size](https://docs.docker.com/subscription-billing/desktop-license/).

2. ###### Security
Podman allows its containers to be [rootless](https://developers.redhat.com/blog/2020/09/25/rootless-containers-with-podman-the-basics#). What is rootless? It means there isn't one central process (daemon) that's in charge of all your containers. Not one central process that could crash and bring down all your containers, not one central process that could get compromised, and start acting maliciously. To be fair, docker [has rootless mode now too](https://docs.docker.com/engine/security/rootless/), so if rootless is your only attraction, just switch to that. 

3. ###### An easier path into Kubernetes and SystemD
The name "Quadlet" was humorously inspired by the idea of ["squashing a Kubernetes kube"](https://archive.fosdem.org/2025/schedule/event/fosdem-2025-5383-running-containers-under-systemd-exploring-podman-quadlet/#:~:text=The%20name%20comes,supported%20to%20deploy), as that was the main intent behind Podmans's [Quadlet]() feature. In other words, bring Kubernetes "Pods" without all the complexity that usually comes with Kubernetes. 

Quadlets facilitate integrating with [systemd](https://systemd.io/) as well. Why bother integrating with systemd?  Your web app is probably running on a server with Linux, and that version of Linux is probably using systemd. Systemd isn't technically required in Linux, but its the unofficial standard task scheduler for most Linux distros, including [Fedora](https://fedoraproject.org/), [Ubuntu](https://ubuntu.com/), [Arch](https://archlinux.org/), and many others. Unless your server is running [Slackware](http://www.slackware.com/config/init.php), integrating with systemd is a no-brainer.

4. ###### Pods
As its name implies, Podman works with sets of [pods](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/8/html/building_running_and_managing_containers/assembly_working-with-pods_building-running-and-managing-containers). Although Docker has no equivalent feature, pods aren't specific to Podman either, as [it took the concept from Kubernetes](https://kubernetes.io/docs/concepts/workloads/pods/). Pods allow containers to be grouped together and given their own network namespace. Once containers are grouped into "pods", those pods can be started/restarted together, like `podman pod start my-app`.

This feature is less geared towards developers, more-so system admins/devops who manage lots of containers. That said, they're easy to setup, and alleviate some of tedium associated with spinning up/down large swaths of containers.

{% end %}

I've been using Podman for a few years now, and overall I've found it a completely acceptable replacement for Docker. That said, I've found most of the tutorials/resources(and subsequently, AI) have described Podman from a more system-admin/devops workflow, not a web developer workflow. So in the next few posts, I'm going to outline some ways I've found to work with Podman containers. Some of them will be fairly familiar (like [podman-compose](https://docs.podman.io/en/latest/markdown/podman-compose.1.html), while others are much newer (like a [kubernetes-style "kube" file](https://www.redhat.com/en/blog/multi-container-application-podman-quadlet)).
