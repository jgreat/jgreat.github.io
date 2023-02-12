---
layout: post
title:  "Rancher Application Pipeline"
date:   2016-02-21 13:33:17 -0600
categories: rancher docker containers cattle ci/cd
---

<a href="http://rancher.com">Rancher Labs</a> has invited me to give a demo during their Web Meetup on Wednesday 07/27/2016. I'm presenting the Application Pipeline we have build at LeanKit. Github, Drone CI and Rancher.

Disclaimer: A lot of this code is specific to how WE do things. It may or may not work out of the box for you. YMMV

### Presentation

- <a href="https://youtu.be/6vtY_4vNvpE?t=4933">Video - My Part starts at about 1:23:00</a>
- <a href="http://jgreat.me/wp-content/uploads/2016/07/Rancher-Application-Pipeline-Demo.pdf">Rancher-Application-Pipeline-Demo</a>

### Demo Code

- <a href="https://github.com/jgreat/vote-demo-web">vote-demo-web</a>
- <a href="https://github.com/jgreat/vote-demo-worker">vote-demo-worker</a>
- <a href="https://github.com/jgreat/vote-demo-results">vote-demo-results</a>

### Example Rancher-Catalogs

These are built using the drone-rancher-catalog Drone plugin.

- <a href="https://github.com/jgreat/rancher-vote-demo-web">rancher-vote-demo-web</a>
- <a href="https://github.com/jgreat/rancher-vote-demo-worker">rancher-vote-demo-worker</a>
- <a href="https://github.com/jgreat/rancher-vote-demo-results">rancher-vote-demo-results</a>

### Drone CI

- <a href="https://github.com/drone/drone">Drone CI Project</a>
- <a href="https://github.com/jgreat/drone-rancher-catalog">drone-rancher-catalog</a> - Drone plugin to build custom rancher catalogs.
- <a href="https://github.com/LeanKit-Labs/drone-cowpoke">drone-cowpoke</a> - Drone plugin to kick off a Rancher Stack upgrade via Cowpoke Service.
- <a href="https://www.npmjs.com/package/buildgoggles">buildgoggles</a> - NPM module to create more descriptive tags for our builds.

### Cowpoke

- <a href="https://github.com/LeanKit-Labs/cowpoke">cowpoke</a> - Connector Service that scans Rancher Environments and upgrade stacks.
