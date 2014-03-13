#Docker Quicktip #1: Entrypoint

#Docker Quicktip #1: Entrypoint

***

The first tip is aptly named “Entrypoint”. In this tips I kind of expect that you’ve played around with Docker a bit, probably even have some containers running for your dev environment. So, in short, if you haven’t played yet, go play and come back!


"Entrypoint" 这个词本身还具有“切入点”的意思，正好用来做我们介绍的第一条技巧。本文中假设你已使用 Docker 一段时间，如果你自己的开发环境中正运行着一些 Docker 容器那就更好了。总之，你要对 Docker 有一定的了解，如果没用过 Docker 的话还是先试用一下再来看这篇文章吧。

***

Entrypoint is great. It’s pretty much like CMD but essentially let’s you use re-purpose CMD as runtime arguments to entrypoint. For example…

Entrypoint 非常好用。它非常像 CMD ，不同的是它是在容器开始运行时把命令作为运行时参数传入到容器中。例如：

***

Instead of:

之前运行命令可能是：

`docker run -i -t -rm busybox /bin/echo foo`
***

You can do:

有了 Entrypoint，你可以这样：

`docker run -i -t -rm -entrypoint /bin/echo busybox foo`

***

This sets the entrypoint, or the command that is executed when the container starts, to call /bin/echo, and then passes “foo” as an argument to /bin/echo.

其实 Entrypoint 就是容器开始运行时执行的命令。因此上面的语句就是让 Docker 容器在开始运行时把 "foo" 作为参数来调用 /bin/echo 。

***

Or you can do, in a Dockerfile:

你也可以在 Dockerfile 中指定 Entrypoint：

    FROM busybox
    ENTRYPOINT ["/bin/echo", "foo"]

    docker build -rm -t me/echo .
    docker run -i -t -rm me/echo bar

***

This passes bar as an additional argument into /bin/echo foo, resulting in “/bin/echo foo bar”

这样，就把"bar"作为额外的参数传递给了`/bin/echo foo`，最终相当于执行`/bin/echo foo bar`

***

Why would you want this? You can think of it as turning CMD into a set of optional arguments for running the container. You can use it to make the container much more versatile. This will lead into the next tip “Exec it”

为什么要用 Entrypoint 呢？可以认为，通过 Entrypoint，我们可以把 Docker 的命令变成一组可选的参数来运行容器。通过它，可以让容器变得更加灵活多用。由此我们引出下一个技巧："Exec it"。

***

