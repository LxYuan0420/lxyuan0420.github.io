---
layout: post
title: "Fix: Overcomming SSH Connectivity Issues with GitLab"
description: "This article describes how to fix the issue of being unable to git clone from GitLab"
---

So, I recently faced a hiccup while trying to clone a company project from GitLab. Here's the error message my terminal threw:

{% highlight bash %}
root@lx:~/work# git clone git@gitlab.com:<project-name>/<repo-name>.git
Cloning into <repo>...
socket: Address family not supported by protocol
ssh: connect to host gitlab.com port 22: Address family not supported by protocol
fatal: Could not read from remote repository.

Please make sure you have the correct access rights
and the repository exists.
{% endhighlight %}

For a bit of context, I was SSH'd into a cloud machine when I tried to git clone a repo from GitLab. The issue likely arose because port 22, the default port for SSH, was blocked. Many cloud service providers block port 22 as a security measure. It's a common practice to prevent unauthorized SSH traffic, which can mitigate risks like brute force login attempts.

But here's the silver lining: I discovered that GitLab.com runs an alternate SSH server on port 443, which is typically open since it's used for HTTPS traffic. If you ever encounter this issue, here's a quick solution. Adjust your ~/.ssh/config to modify your connection settings to GitLab.com:

{% highlight bash %}
Host gitlab.com
  Hostname altssh.gitlab.com
  User git
  Port 443
  PreferredAuthentications publickey
  IdentityFile ~/.ssh/id_ed25519
{% endhighlight %}

And there you go! Problem solved. I hope this note assists someone facing a similar issue. Happy coding! 🚀

Related:
- [GitLab.com now supports an alternate git+ssh port](https://about.gitlab.com/blog/2016/02/18/gitlab-dot-com-now-supports-an-alternate-git-plus-ssh-port/)
- [Getting permission denied (public key) on gitlab](https://stackoverflow.com/questions/40427498/getting-permission-denied-public-key-on-gitlab)
- [Gitlab can't clone repository even though ssh works](https://stackoverflow.com/questions/33837103/gitlab-cant-clone-repository-even-though-ssh-works)
- [ssh: connect to host gitlab.com port 22: Network is unreachable](https://stackoverflow.com/questions/65299117/ssh-connect-to-host-gitlab-com-port-22-network-is-unreachable)
