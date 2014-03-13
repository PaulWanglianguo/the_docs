#Quickly SSH into a Docker container
#如何用 SSH 快速登录 Docker 容器

I’ve been playing around with Docker for a few months and it’s great.

玩了几个月的 [Docker](https://www.docker.io/) 发现这东西实在太棒了！

One thing that is slightly rage-inducing when using docker is having to SSH into a container. Generally if you have multiple containers that run sshd, you’ll want the SSH ports to be generated randomly to prevent conflicts. Connecting via SSH is slightly tedious to do in this case, as there are several steps involved.

但是每次使用 [Docker](https://www.docker.io/) 都必须通过 SSH 登陆到容器令人有点儿抓狂。一般来说如果有多个容器都在跑 SSHD 服务，就得让 SSH 随机生成端口以防止冲突。用 SSH 登陆需要多个步骤，在多个 SSHD 服务一起运行的时候就更加繁琐了。


###Problem
###问题
***

Basically the routine is to run docker ps:

用 SSH 登陆的基本思路就是使用 `docker ps`:


    CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                                                NAMES
    e53a722096f0        aafd21071b99        /opt/bin/run_all    41 minutes ago      Up 28 minutes       127.0.0.1:49251->22/tcp, 127.0.0.1:49252->9200/tcp   es-b
    0208431f9c7b        aafd21071b99        /opt/bin/run_all    41 minutes ago      Up 28 minutes       127.0.0.1:49249->22/tcp, 127.0.0.1:49250->9200/tcp   es-a

If you scroll to the right of that you’ll see the randomly generated port in the PORTS column:

以上即是 `docker ps` 的输出结果，在最右面的 PORTS 这一列可以看到随机生成的端口号：

    127.0.0.1:49251->22/tcp


The next step is to type out the SSH command. You can either give in to RSI by copying the port with the mouse, or type it out:

接下来就是输入 SSH 命令，可以用鼠标复制端口号，或者直接输入端口号：

    ssh root@localhost -p 49251

I think that’s too much work to simply SSH into a container!

仅仅是通过 SSH 登录容器而已，这也太麻烦了！

###Solution
***


I gave into annoyance and wrote a straightforward Python script to automatically do this for me. The Python script is calleddssh. It runs docker inspect, grabs the port, then runs SSH for you:

我怒了！于是我写了个 Python 脚本来解决问题。Python 脚本叫做 `dssh`，脚本里面先执行 `docker inspect`，然后从结果中取出端口，最后再调用 SSH 登录容器：
    
    #!/usr/bin/env python
    import sys
    import subprocess
    import json
     
    cmd = 'sudo docker inspect {}'.format(sys.argv[1])
    output = subprocess.check_output(cmd, shell=True).decode('utf-8')
    data = json.loads(output)
     
    port  = data[0]['NetworkSettings']['Ports']['22/tcp'][0]['HostPort']
     
    cmd = 'ssh root@localhost -p {}'.format(port)
    subprocess.call(cmd, shell=True)

You might notice it makes some assumptions which will crash the script if they’re not followed. There needs to be an argument for the container ID or name. In the example above I could use either "e53a722096f0" or "es-b" as the argument. There also has to be an sshd process port listening on 22, and a configured container to forward that port.

也许你注意到了，这个脚本中我们假设的一些条件如果不满足，脚本就会崩溃。我们需要一个参数来表示容器的 ID 或者名称。在上面的例子中我可以指定参数为 "e53a722096f0"或 者 "es-b"。同时还必须有一个 SSHD 进程监听 22 端口，还要配置一个容器来转发 22 端口。

The output of docker inspect is a handy JSON string coming straight from the Docker Remote API. This in turn gives us the port we need. We then simply run subprocess.call and suddenly you’re in the container.

`docker inspect` 命令的结果是由 [Docker Remote API](http://docs.docker.io/en/latest/api/docker_remote_api_v1.8/#inspect-a-container) 输出简单易读的 JSON 字符，结果中也返回来我们需要的端口号。有了端口号，我们只需要调用 `subprocess.call` 来执行 SSH 命令，然后瞬间登陆到容器。


###Closing
###结束语
***
I did attempt this not long ago but failed. The attempt was to use docker-py but unfortunately it seemed to need root, which I couldn’t work a way to avoid it. I needed to SSH into docker as my non-privileged user because of SSH keys, so instead I made it print out the SSH command for it to be executed, but it was a cumbersome solution.

不久之前我试这个方法没有成功。方法中使用了 [`docker-py`](https://github.com/dotcloud/docker-py)，必须用 root 权限才能执行，所以我只能用非 root 用户来登陆 Docker 容器。所以我在脚本中把 SSH 命令打印出来执行，但是这样做显得太笨拙了。


If you want the latest version, it’ll be updated in my dotfiles repository on github.

请去我 Github 上的 [dotfiles仓库](https://github.com/gak/dotfiles/blob/master/home/bin/dssh) 查看最新的脚本。
2014年2月2日
Gerald Kaszuba.

This entry was posted in Hacking and tagged docker, python, ssh on January 2, 2014 by Gerald Kaszuba.
