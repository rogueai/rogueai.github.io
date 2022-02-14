+++
title = "`DRAFT` Self hosted Gitea on Sinology"
date = 2022-02-14
description = "Setup a self-hosted instance of Gitea on Synology DiskStation"
draft = true
toc = false
author = "rogueai"
categories = ["post"]
tags = ["selfhost", "gitea", "synology"]
license = "cc-by-nc-sa-4.0"
+++

- (syno) run chown 1000:1000 on /data folder
- create ssh key gitea, gitea.pub
- add and verify ssh key
- (syno) open firewall 2222
- add config ~/.ssh/config for host and user: git
- make sure to generate ssh keys accepted by your client, newer versions of openssl
  deprecated rsa algorithm
- debug ssh with -vT to troubleshoot issues
  `ssh -vT git@<host> -p 2222`