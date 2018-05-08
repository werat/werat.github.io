---
layout: post
title:  "Happy ssh agent forwarding for tmux/screen"
date:   2017-02-04 11:55:00 +0300
categories: tmux ssh ssh-forwarding
---

If you often use ssh+tmux combination and ssh keys forwarding, you've definitely been in an unpleasant situation:

* connect to some remote machine via ssh and create a tmux session
* use it happily
* detach from tmux and disconnect from server
* connect again (e.g. next day) and attach to the tmux session
* push something to git (or connect to another server)...
* ...
* `Permission denied (publickey).`

Let's discuss why this is happening. The magic behind ssh forwarding is quite simple: ssh-agent creates a socket (on linux usually something like `/tmp/ssh-hRNwjA1342/agent.1342`) which is used by other applications to communicate with ssh-agent. Obviously these applications need a way to find this socket, so ssh-agent also sets an environment variable `SSH_AUTH_SOCK`, which contains path to the socket.

{% highlight bash %}
> env | grep SSH_
SSH_AUTH_SOCK=/private/tmp/com.apple.launchd.4ymgbbAxhL/Listeners
{% endhighlight %}

When you create a new tmux session environment variables from current shell are captured and later used for every new window/pane. Thus all your panes inside tmux will have the same value of `SSH_AUTH_SOCK`. When you disconnect from server ssh connection terminates and socket is deleted. When you reconnect *a new socket is created* (and `SSH_AUTH_SOCK` now points to it), but all your panes inside tmux *still have the old value*.

What first comes to mind is to find the path of a new socket and export it in broken panes. You can totally do it, but this is pain. For new panes we can tell tmux to reload environment variables. To do so just add the following lines to your `~/.tmux.conf` file:

{% highlight bash %}
set -g update-environment "SSH_AUTH_SOCK"
{% endhighlight %}

This is clearly a half measure, because you'll still have all your existing panes being broken every time the ssh connection dies. We might want to modify `SSH_AUTH_SOCK` of existing panes, but there is no good (or even bad, but portable) way to do it. Luckily there is another way.

As I said, `SSH_AUTH_SOCK` points to the current ssh-agent socket and this socket dies, when the connection is terminated. The last thing is a problem, and all problems in computer science can be solved by another level of indirection. Let's point `SSH_AUTH_SOCK` to *a symlink* which is pointing to a ssh-agent socket; this way all panes will have a valid value in `SSH_AUTH_SOCK`. For `tmux` this can be done also via config[^1]:

{% highlight bash %}
> cat ~/.tmux.conf
set-environment -g 'SSH_AUTH_SOCK' ~/.ssh/ssh_auth_sock
{% endhighlight %}

If you prefer `screen` (which is weird), you should use `~/.screenrc`[^2]:

{% highlight bash %}
> cat ~/.screenrc
setenv SSH_AUTH_SOCK ~/.ssh/ssh_auth_sock
{% endhighlight %}

The only problem is to maintain the symlink valid. This is where `~/.ssh/rc` comes out. This script is invoked by the SSH server for each incoming SSH connection[^3].

{% highlight bash %}
> cat ~/.ssh/rc
if [ -S "$SSH_AUTH_SOCK" ]; then
    ln -sf $SSH_AUTH_SOCK ~/.ssh/ssh_auth_sock
fi
{% endhighlight %}

Great, now for every new ssh connection (and if ssh forwarding is enabled) we'll update the symlink to the new value. And everything will work fine until this connection is terminated.

I personally lived with this solution for quite a long time, until I've started to use lsyncd[^4] for syncing my source code with remote machines. The thing is lsyncd uses rsync for syncing files and this means it creates new ssh connections once in a while. The problem is that when a new connection is created *it updates your symlink*. And when it dies the symlink becomes broken.

To fix this problem I told `/.ssh/rc` not to modify the symlink *if current is still alive*. This way new ssh connections will not touch the symlink until the connection which created it is still alive.

{% highlight bash %}
> cat ~/.ssh/rc
if [ ! -S ~/.ssh/ssh_auth_sock ] && [ -S "$SSH_AUTH_SOCK" ]; then
    ln -sf $SSH_AUTH_SOCK ~/.ssh/ssh_auth_sock
fi
{% endhighlight %}

# Summary

1. Setup the `~/.ssh/rc` to maintain the `~/.ssh/ssh_auth_sock` symlink alive
2. Setup the `~/.tmux.conf` to make `SSH_AUTH_SOCK` point to the `~/.ssh/ssh_auth_sock`
3. ...
4. Profit

# Links
[^1]: http://man7.org/linux/man-pages/man1/tmux.1.html#ENVIRONMENT
[^2]: https://www.gnu.org/software/screen/manual/screen.html#Setenv
[^3]: http://docstore.mik.ua/orelly/networking_2ndEd/ssh/ch08_04.htm
[^4]: https://github.com/axkibe/lsyncd
