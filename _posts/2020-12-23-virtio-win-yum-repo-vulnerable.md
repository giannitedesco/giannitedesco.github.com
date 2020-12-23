---
title: Anyone who has ever installed virtio-win by yum is vulnerable
layout: post
comments: true
mathjax: true
date: 2020-12-23 12:40:00 +0900
tags: security vuln cve virtio-win fedora redhat cve-2020-29665
summary: In which we explain the evils of gpgcheck=0
---

If you have _ever_ installed the `virtio-win` package in accordance with the
[instructions from
fedora](https://docs.fedoraproject.org/en-US/quick-docs/creating-windows-virtual-machines-using-virtio-drivers/index.html),
then your system is vulnerable to a remote root exploit every time you do a yum
upgrade, and will stay like that forever more, until you remove the virtio-win
yum repo.

That is because the instructions tell you to setup a yum repo with the with a
plain http `baseurl` and `gpgcheck=0`... This, essentially, disables all
security and allows a network man-in-the-middle (even a regular old web proxy)
to inject bogus updates every time you do a `dnf upgrade` - which will be
nightly if you have installed `dnf-automatic`.

This entire process, including exploitation, can happen without dnf providing
any hint to the user that it is downloading arbitrary code insecurely over the
internet and running it.

# What can you do about it?
Turns out that there is not a whole lot you can do about this yet. My best
advice to you is to disable this yum repo immediately.

I reported this bug [in
September](https://bugzilla.redhat.com/show_bug.cgi?id=1878594) but it turns
out it had already been [reported in January
2016](https://bugzilla.redhat.com/show_bug.cgi?id=1353036). 

There is [a github
issue](https://github.com/virtio-win/virtio-win-pkg-scripts/issues/24) tracking
the change required to sign the contents of virtio-win repository.

# Windows has a better security culture than Linux
Note, that the windows drivers themselves are signed. That's because loading
windows drivers without signatures is a complete rigmarole so the drivers would
be next to useless if they weren't signed.

But here, the signatures are luring people in to a false sense of security
because you may imagine that since all RPM is doing is downloading some opaque
blobs which only a guest will install, that as long as the blobs are signed and
the windows guest OS verifies them then an end-to-end root of trust is
established.

But this is not so, since the Linux host is compromised. All an attacker has to
do is create a new RPM based on the old virtio-win RPMs (including all the
valid signatures) and just add a `%postinstall` RPM script which roots the
host. So now it doesn't matter that the guest is secure, because the host is
compromised. And if the host is compromised then all bets about the integrity
of the guest are off.

If RPM applied its signatures with the same dilligence as Windows update, we
probably would not be in this mess.

I also filed [a bug against
dnf](https://bugzilla.redhat.com/show_bug.cgi?id=1878595) that it insecurely
installs code from the internet without so much as a warning message.

# DNF does not check TLS certs if you use https!
Initially I thought that a quick workaround would be to change the repo URL to
https. After all, `fedoraproject.org` should be trustworthy.

But according to the DNF developers "DNF is happy with any HTTPS connection,
even with a self-signed certificate and a complete redirect to a different
server would be unnoticed."

Apparently there is a plan to solve this but it won't land before RHEL9 because
the user experience would necessarily change if dnf were to check HTTPS
certificates.

Of course, there are good historical reasons not to use TLS, it breaks
caching. But given todays threat landscape, mirror services may just want to
suck up this cost.

# Arbitrary remote code execution is arbitrary remote code execution
Merry christmas, happy new year, it'll soon be 2021, and here's to another year
of telling people that arbitrary remote code execution is arbitrary remote code
execution.

You can refer to the virtio-win repo issue as `CVE-2020-29665`.
