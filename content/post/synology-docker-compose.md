+++
title = "Docker Compose on Synology NAS"
date = 2022-02-11
description = "Deploy docker containers on Synology NAS using Docker Compose"
draft = false
toc = false
author = "RogueAI"
categories = ["self host"]
tags = ["post"]
[[copyright]]
  owner = "RogueAI"
  date = "2022"
  license = "cc-by-nc-sa-4.0"
+++

A quick howto with the steps required to setup `docker` and `docker-compose` on Synology NAS:
- Install Docker package
- Make sure the docker network is allowed through the firewall:
  - Navigate to: Control Panel -> Connectivity -> Security -> Firewall
  - Add a custom rule
    - Ports: Add all your exposed container ports
    - Protocol: TCP
    - Source IP: Specific IP -> Specify your docker network's IP and subnet mask
- Create a scheduled task
  - Navigate to: System -> Task Scheduler
  - Create a new task for user `root`
  - In Task Settings, set a User-defined script
    ```bash
    docker-compose -f /volume1/docker/docker-compose.yml up -d
    ```
    Change the path to where your docker compose file is located
  - Set the task as disabled, and run it manually from the context menu