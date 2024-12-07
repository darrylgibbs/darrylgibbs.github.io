---
title: Ansible All Distro Docker Install
author: darryl gibbs
date: 2024-12-06 08:50:49 -03:00
tags: [docker, ansible, playbooks]
categories: [homelab, tutorial]
---

As part of my SysAdmin journey, I've begun to play with Ansible.  For a while I've been using very simple playbooks to update my homelab's VMs and LXC containers, but it's time to take that a bit further.


## Desired Outcome

The goal today is to start using playbooks in a modular way, thus allowing more flexibility and the reuse of playbooks across multiple projects.

The idea here, is to have one main playbook that determines the OS on the host, namely Debian, Ubuntu, or a RedHat derivative OS (such as CentOS, Rocky, Fedora etc.)  

From there, a chain of appropriate `task.yml` files will begin updating the host OS, installing Docker, and finally copying over a test webpage and starting up an Nginx Docker container to serve the webpage.

If we see the image below, we're golden.

{{< image src="/" alt="Nginx webpage test" position="center" style="border-radius: 8px;" >}}


## Playbooks and task.yml files


## Technical Difficulties


## Lessons Learned