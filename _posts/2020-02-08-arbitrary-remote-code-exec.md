---
title: Arbitrary Remote Code Execution is (Still) Arbitrary Remote Code Execution
layout: post
comments: true
mathjax: true
date: 2020-02-08 10:14:00 +0900
tags: security containers lxc snapcraft docker
summary: Where we warn the youth about downloading malware and running it
---

Long gone are the days when servers were like little pets. You logged in to
them, you stroked them, you ran little commands on them. Probably some commands
you copied and pasted out of stackoverflow or that you found in some google
search results.

Then when the server died, you had a little funeral, buried it in a little pet
cemetery, and then tried to remember how the hell you had set it up.

But that was fine, you were in grief, you didn't want another pet to be
identical to your dead pet anyway.

That is, Unless you were trying to deliver consistent, performant, and
uninterrupted service to colleagues, customers, or the like.

## Cattle, not Pets
In that case you'd want something a bit more repeatable. A bit more automated.
A bit more like a production-line eviscerating pigs for the manufacture of
sausage.

First there were configuration management systems, like:
- 1993: CFEngine
- 2000: Spooner (proprietary, developed by yours truly) :)
- 2005: Puppet
- 2009: Chef
- 2012: Ansible

These tools allowed you to represent the configuration or desired state of a
server as code or data but, in either case, as a version-controlled repository
subject to the quality-oriented practices which began to be adopted by
programmers in the early decades of the 21st century.

Later, in 2013, Docker came along. The idea behind docker was to use containers
as a way to build or deploy software in an industrialised fashion, like the
afore-mentioned sausage factory. The more inspirational, but frankly less
tasty, metaphor here would be the containerisation of bulk cargo. Before this
idea took hold, containers were thought of as more to do with the containment
or confinement of running software. They were sometimes called jails and still
widely viewed as being fancy chroots.

In combining the underlying technology of namespaces, control-groups, jails,
containers with the philosophy and concepts of infrastructure-as-code. Docker,
and systems like it, revolutionised how software services were delivered. By
now these, and related, techniques are ubiquitous in the data-centre.

Which is great, because now all of the insecure and insane practices get
written down and permanently archived in git repositories instead of being a
secret burden of guilt and shame carried on the shoulders of an entire class of
people: systems administrators.

Now the blame can be diffused among software developers of all hues and stripes
and we can call it a cultural problem :)

## Building and Bootstrapping Infrastructure Now.
Well, a container really is just a fancy chroot. Whereas chroot changes the
processes view of the VFS namespace by setting the root to some directory,
thereby restricting its view of the filesystem to a subtree of that directory.
Containers are processes which have their own private, possibly unique, set of
mounts.

However, we still need to populate the mounted directories or filesystems with
the software that's going to run!

There are any number of ways of doing this.

## SnapCraft
[Snapcraft](https://snapcraft.io) uses a YAML file which defines sources to be
downloaded, which will then be compiled and installed on top of a base OS layer
which contains compilers, a package manager, etc.

Snapcraft provides security measures, such as the ability to check source code
downloads against checksums with the `source-checksum` feature and the ability
to download code over https.

However, if you just download code from the internet over http, then you're
just an [ettercap](https://github.com/Ettercap/ettercap) or a dns poisoning
away from being fed exploit code which will automatically be run as part of the
snap build process.

The snapcraft tour, part of the getting started guide, suffered from this
problem, which was
[reported in Oct 2016](https://bugs.launchpad.net/snapcraft/+bug/1634415)
and was [fixed in May 2017](https://github.com/snapcore/snapcraft/pull/1329).

Which is great. We shouldn't be teaching a new generation of programmmers to
automatically trust and execute arbitrary remote code.

## LXC Templates : CVE-2017-18641
[LXC](https://linuxcontainers.org) shipped a series of templates which are
scripts which initialise the container root filesystem.

The centos and fedora templates relied on yum being installed on the host. The
hosts yum would be used to download RPMs and install them in to the rootfs.

The RPMs were being downloaded over http, which could be okay, since RPMs are
signed. However yum was being invoked with `--nogpgcheck` which disabled this
feature. The upshot being that anyone using the template is also an ettercap or
a hacked mirror or untrustworthy proxy away from having arbitrary code executed
as root.

This was [reported](https://bugs.launchpad.net/ubuntu/+source/lxc/+bug/1661447)
on Feb 3 2017. And it turned out after a quick investigation that the problem
applied to around a dozen of the template scripts.

Some were fixed in short order. But there were so many cases of this, all in
different ad-hoc scripts, by different authors, that it became "a bit of a
mess."

By Feb 2 2018 the LXC team had started work on
[distrobuilder](https://github.com/lxc/distrobuilder) which had support for
https and gpg from the outset and would be difficult to get wrong.

By LXC 3 the templates system had been removed to a historical repo and the
_status quo_ is now that images are built securely and signed on trusted
infrastructure and then sent to users over https and with a signature which
is checked by `lxc-download`.

This is a fine example of the concerns which go in to building a software
distribution mechanism in which trust can be established.

## Insecurely Downloaded Remote Code is, by Definition, Arbitrary.
### And If it's Arbitrary then You Have No Reason to Trust it.
#### So Why Would You Want to Execute it?

It's certainly tempting, when writing a Dockerfile, to imagine that everything
is executing in some sort of secure isolation layer and that, because of this,
security is irrelevant.

For now, let's ignore the historical record security vulnerabilities in
container isolation, and the entire available attack surface of the kernel, and
the fact that namespaces and control groups are not purpose-built security
isolation systems. And the fact that even if they were such, then with all their
flexibility and configurability there would bound to be many possible insecure
configurations. Let's ignore all of that for now.

Usually, if you're creating a container, you want to be able to trust the code
inside it with whatever else is in that container. So even if an attacker can't
break out to the host, you won't want them to have full control over the
container because they may be able to deface your website, defraud your
customers, introduce backdoors in to your software builds, etc.

And if the attacker has corrupted the image at container build time. Then you
don't even have the defence that the container instances are cattle that can be
shot in the head with a bolt-gun and replaced with a fresh new lump of meat,
cloned from the same DNA.

So are these practices common?

From a quick search on github I was able to find
[23,000 instances](
https://github.com/search?l=&q="wget+http%3A%2F%2F"+language%3ADockerfile&type=Code) of "wget http://..." in Dockerfiles.

By randomly clicking them you can see that most of these (100% of the ones I
looked at anyway) absolutely are downloading arbitrary code and then running
it.

I was also able to find a further [4,000 instances](
https://github.com/search?utf8=✓&q="curl+http%3A%2F%2F"+language%3ADockerfile&type=Code&ref=advsearch&l=&l=)
for curl, but some of these looked legitimate.

With a bit of regex trickery one could probably find many more instances.

GitHub really is a repository of endless trivial security vulnerabilities.

## A Brief Message from the Cast
While software vendors take vulnerabilities seriously when they are discovered.
And act to remediate them in a timely and responsible fashion. We probably need
to be doing a little bit more, as a community, for the typical users of these
things to drill in to them the dangers involved.

It's never going to be comfortable for purveyors of band-saws to be always
pointing out that band-saws can irreparably alter the relation between a user's
fingers and said user's hands.

That's just not the fun stuff, compared to all the cool things you can make and
do with power-tools.

However, when we see how widespread some of these unsafe practices are, we
might want to think about what sort of technical or social interventions might
work to improve the situation.
