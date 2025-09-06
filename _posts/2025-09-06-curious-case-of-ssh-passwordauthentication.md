---
layout: post
title: 'The Curious Case of SSH PasswordAuthentication'
date: '2025-09-06 11:00:00 +1000'
categories: linux
---
# The Curious Case of SSH PasswordAuthentication

I recently got an old Intel NUC and, as any developer would do, I decided to use it as a box to run my fun experiments on. So I installed Proxmox on it, spun up Ubuntu 22.04, and, as you’d expect, added my private SSH keys. Then I decided to disable password-based authentication in favor of public key authentication. So I ran:

```bash
$ sudo vi /etc/ssh/sshd_config
```

and changed these lines before saving:

```text
PasswordAuthentication no
ChallengeResponseAuthentication no
PubkeyAuthentication yes
PermitRootLogin no
```

Then I ran:

```bash
$ sudo systemctl restart ssh
```

Simple, right? But when I tried connecting to the server without ssh keys, it still asked me for my password! It had been a while since I last did this, so I was wondering if anything had changed?

I logged back into the server to double-check if I had modified `sshd_config` or `ssh_config`, and it looked like I had edited the right file. So hang on, had `sshd` actually reloaded the new values?

I restarted the service again and ran:

```bash
$ sudo sshd -T | grep -E 'passwordauthentication'
passwordauthentication yes
```

Seeing `passwordauthentication` still set to yes confused me, since this should have been a very simple change! One part of me really wanted to ask ChatGPT about this, but the other part of me really wanted to figure it out myself. I started wondering if there was any environment variable or another file that could override this. I decided to run a grep query and voilà!

```bash
$ sudo grep -Rni passwordauthentication /etc
/etc/ssh/sshd_config:66:PasswordAuthentication no
/etc/ssh/sshd_config:88:# PasswordAuthentication.  Depending on your PAM configuration,
/etc/ssh/sshd_config:92:# PAM authentication, then enable this but set PasswordAuthentication
/etc/ssh/ssh_config:25:#   PasswordAuthentication no
/etc/ssh/sshd_config.d/50-cloud-init.conf:1:PasswordAuthentication yes
```

There it was: `PasswordAuthentication` was set to `yes` in `/etc/ssh/sshd_config.d/50-cloud-init.conf`. Could this file be overriding my changes?

I decided to delete this file since that was the only thing it was setting. Then I restarted the service and checked what sshd was loading for `PasswordAuthentication`:

```bash
$ sudo systemctl restart ssh
$ sudo sshd -T | grep -E 'passwordauthentication'
passwordauthentication no
```

And that worked! So the `50-cloud-init.conf` file was indeed overriding my changes in `/etc/ssh/sshd_config`.

Later, I researched more and found out that while the ability to include extra config files has existed in OpenSSH since version 7.3 (2016), what changed is that Ubuntu 22.04 started shipping with `/etc/ssh/sshd_config.d/50-cloud-init.conf` by default, and that file explicitly sets `PasswordAuthentication yes`. That’s why I didn’t remember running into this on older Ubuntu versions.

If you open `/etc/ssh/sshd_config`, you can see where it includes `.conf` files from `/etc/ssh/sshd_config.d`:

```text
Include /etc/ssh/sshd_config.d/*.conf
```

Deleting the file fixed it for me, but a cleaner approach might be to edit `/etc/ssh/sshd_config.d/50-cloud-init.conf`.

In the end, I was happy that I had solved the issue, even if what should have been a simple change ended up taking a chunk of my morning.