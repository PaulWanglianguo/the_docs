# PaaS under the hood, episode 1: kernel namespaces
# PaaS的底层实现1：内核命名空间
Posted on November 28, 2012 by Jérôme Petazzoni

Making things simple is a lot of work. At dotCloud, we package terribly complex things – such as deploying and scaling web applications – into the simplest possible experience for developers. But how does it work behind the scenes? From kernel-level virtualization to monitoring, from high-throughput network routing to distributed locks, from dealing with EBS issues to collecting millions of system metrics per minute… As someone once commented, scaling a PaaS is “like disneyland for systems engineers on crack”. Still with us? Read on!

把事情变得简单是一件繁重的工作。在dotCloud，我们把像部署，扩展Web应用程序这样超级复杂的事情，打包成对于开发者来说最简单的体验。但我们是如何做到这点的呢？从内核层级的虚拟化到监控，从高吞吐量的网络路由到分布式锁，从处理EBS问题到每分钟收集无数的系统指标……有人说，扩展一个PaaS是一个不可能的任务，对我们来说是这样么？继续往下看吧！

This is the 1st installment of a series of posts exploring the architecture and internals of platorm-as-a-service in general, and dotCloud in particular. For our first episode, we will introduce namespaces, a specific feature of the Linux kernel used by the dotCloud platform to isolate applications from each other.

这个系列的文章探索了一般认为的PaaS的架构和内部实现，以dotCloud平台为示例。在这第一篇文章中，我们会介绍名称空间（namespaces），这是一个dotCloud平台用来隔离应用程序的Linux内核的特性。

## Part 1: Namespaces

The first time I was introduced to Linux Containers (LXC), I got the (very wrong) impression that they relied mainly on control groups (or “cgroup” in short). It’s an easy mistake to make: each time you create a new container named e.g. “jose”, a cgroup of the same name appears, e.g. in “/cgroup/jose”. But actually, even if control groups are useful to Linux Containers (as we will see in part 2), the really important infrastructure is provided by namespaces.

当我第一次认识 Linux Containers (LXC) 时，我有一个（非常错误的）印象——他们主要依赖于control groups（cgroups）。这是一个很容易犯的错误：当你每次创建一个新的container，比如叫“jose“，就会出现一个同样名字的cgroup，如"/cgroup/jose"。但事实上，虽然cgroups对于Linux Container来说非常有用，但真正重要的底层支持是来自于namespaces。

Namespaces are the real magic behind containers. There are different kinds of namespaces, as we will see; each kind of namespace applies to a specific resource. And, each namespace creates barriers between processes. Those barriers can be at different levels.

Container的隔离主要是由namespaces实现的。下面我们将会看到很多不同的namespaces，他们各自应用于相应的资源，并且从不同的层级将进程隔离开来。

### The pid namespace

This is probably the most useful for basic isolation.

这大概算是最有用的基本隔离了。

Each pid namespace has its own process numbering. Different pid namespaces form a hierarchy: the kernel keeps track of which namespace created which other. A “parent” namespace can see its children namespaces, and it can affect them (for instance, with signals); but a child namespace cannot do anything to its parent namespace. As a consequence:

每一个pid namespace都有自己的进程号。所有的pid namespaces构造成一个层级结构：内核会记录每个namespace的父子关系。父namespace可以看到子namespace，并且可以影响他们（例如，可以向他们发出signals）；但是反之则不行。因此：

- each pid namespace has its own “PID 1” init-like process;
- processes living in a namespace cannot affect processes living in parent or sibling namespaces with system calls like kill or ptrace, since process ids are meaningful only inside a given namespace;
- if a pseudo-filesystem like proc is mounted by a process within a pid namespace, it will only show the processes belonging to the namespace;
- since the numbering is different in each namespace, it means that a process in a child namespace will have multiple PIDs: one in its own namespace, and a different PID in its parent namespace.


- 每一个pid namespace都有自己的，像/sbin/init一样pid=1的进程；
- 每个namespace中的进程不能用类似kill或ptrace这样的系统调用来影响父，或兄弟namespace，因为进程id仅在指定的namespace内部才有意义。
- 如果一个像proc这样的伪文件系统被一个pid namespace中的进程挂载了，/proc目录只会显示这个namespace中的进程。
- 因为每个namespace中的进程编号是独立的，所以一个子namespace中的进程会有多个不同的pid——在自己namespace中的pid和在父namespace中的pid。

The last item means that from the top-level pid namespace, you will be able to see all processes running in all namespaces, but with different PIDs. Of course, a process can have more than 2 PIDs if there are more than two levels of hierarchy in the namespaces.

最后一条意味着顶层的从pid namespace可以看到所有namespace中运行的任意进程，但进程的pid和在子namespace中看到的不同。当然，如果namespace的层级超过2级的话，一个进程也可以有超过2个的pid。

### The net namespace

With the pid namespace, you can start processes in multiple isolated environments (let’s bite the bullet and call them “containers” once and for all). But if you want to run e.g. a different Apache in each container, you will have a problem: there can be only one process listening to port 80/tcp at a time. You could configure your instances of Apache to listen on different ports… or you could use the net namespace.

通过pid namespace，你可以在多个隔离的环境（以后统称容器）中启动进程，但如果你想要在每个容器中运行像Apache这样的网络程序，就会遇到一个问题：一次只能有一个进程监听80/tcp端口。把每个Apache配置为监听不同的端口当然能解决问题……但更简单的方法是，使用net namespace。

As its name implies, the net namespace is about networking. Each different net namespace can have different network interfaces. Even lo, the loopback interface supporting 127.0.0.1, will be different in each different net namespace.

从名字就可以看出，net namespace与网络相关。不同的net namespace可以有不同的网络接口。即使是lo——用于支持127.0.0.1的loopback接口，在每个net namespace中都不相同。

It is possible to create pairs of special interfaces, which will appear in two different net namespaces, and allow a net namespace to talk to the outside world.

我们可以创建一对特殊的网络接口（如eth0），这对接口会出现在两个不同的net namespace，然后允许一个net namespace连接外部的网络。

A typical container will have its own loopback interface (lo), as well as one end of such a special interface, generally named eth0. The other end of the special interface will be in the “original” namespace, and will bear a poetic name like veth42xyz0. It is then possible to put those special interfaces together within an Ethernet bridge (to achieve switching between containers), or route packets between them, etc. (If you are familiar with the Xen networking model, this is probably no news to you!)

一个典型的容器会有自己的loopback接口（lo），和一个特殊的接口，一般叫eth0。eth0的另一端会连接到最顶层的namespace，并且具有一个诗意的名字，如veth42xyz0。然后用桥接的方法把所有的这些eth0和主机连接起来(来实现容器间的数据交换)，和在容器之间路由数据包等等。（如果你了解Xen的网络模型，那你应该对此感到很熟悉！）

######**译者注：docker默认采用veth方式，lxc的5种网络类型可见http://manpages.ubuntu.com/manpages/precise/man5/lxc.conf.5.html

Note that each net namespace has its own meaning for INADDR_ANY, a.k.a. 0.0.0.0; so when your Apache process binds to *:80 within its namespace, it will only receive connections directed to the IP addresses and interfaces of its namespace – thus allowing you, at the end of the day, to run multiple Apache instances, with their default configuration listening on port 80.

要注意的是，每个net namespace都有自己的，对于INADDR_ANY（即0.0.0.0）的定义；所以当你的Apache进程绑定到namespace内部的*:80时，它就只会接收那些指向这个namespace的ip地址和端口的连接。这让你最终能够运行多个Apache实例，并且使用他们监听80端口的默认配置。

In case you were wondering: each net namespace has its own routing table, but also its own iptables chains and rules.

顺便提一下：每一个net namespace都有自己的路由表，自己的iptables的chains和rules。
<<<<<<< HEAD:paas-under-the-hood-episode-1-kernel-namespaces.md

### The ipc namespace

This one won’t appeal a lot to you; unless you passed your UNIX 101 a long time ago, when they still taught about IPC (InterProcess Communication)!

这个部分不会很吸引你，除非你是在很久以前通过你的UNIX基础课程，那时候，他们还在教IPC（InterProcess Communication）！

IPC provides semaphores, message queues, and shared memory segments.

IPC提供信号量（semaphores），消息队列（message queues）和共享内存（shared memory segments）的特性。

While still supported by virtually all UNIX flavors, those features are considered by many people as obsolete, and superseded by POSIX semaphores, POSIX message queues, and mmap. Nonetheless, some programs – including PostgreSQL – still use IPC.

尽管几乎所有的UNIX版本都还支持这些特性，但有有很多人觉得这些特性已经过时，可以被POSIX semaphores，POSIX message queues，和mmap所取代。尽管如此，有一些程序 —— 包括PostgreSQL —— 仍然在使用IPC。

What’s the connection with namespaces? Well, each IPC resources are accessed through a unique 32-bits ID. IPC implement permissions on resources, but nonetheless, one application could be surprised if it failed to access a given resource because it has already been claimed by another process in a different container.

这些和namespaces有什么关系？唔，每个IPC资源都是通过一个唯一的32-bits的ID来访问的。IPC实现了资源的访问控制，但尽管如此，如果你的程序无法使用某个资源，是因为在其他容器中的某个进程使用了同样的资源，这会让人感到很意外。

######**译者注: IPC准确地说不仅实现了资源访问的控制，更是进程间数据交互的通信机制。

Introduce the ipc namespace: processes within a given ipc namespace cannot access (or even see at all) IPC resources living in other ipc namespaces. And now you can safely run a PostgreSQL instance in each container without fearing IPC key collisions!

现在我们引入ipc namespace：在一个ipc namespace中的进程无法访问（甚至看到）位于其他ipc namespace中的IPC资源。现在你可以在每个容器中安全的运行PostgreSQL，而不用担心会发生IPC key冲突了。

### The mnt namespace

You might already be familiar with chroot, a mechanism allowing to sandbox a process (and its children) within a given directory. The mnt namespace takes that concept one step further.

你也许已经很熟悉chroot，chroot是一个可以让进程(及其子进程)在指定目录下以沙箱的方式运行。mnt namespace进一步地发展这个概念。

As its name implies, the mnt namespace deals with mountpoints.

就像名字所暗示的，mnt namespace处理mountpoints的问题。

Processes living in different mnt namespaces can see different sets of mounted filesystems – and different root directories. If a filesystem is mounted in a mnt namespace, it will be accessible only to those processes within that namespace; it will remain invisible for processes in other namespaces.

在不同mnt namespace下的进程可以看到不同的挂载文件系统，和不同的root目录。一个挂载在某个nmt namespace下的文件系统，只能被同一个namespace下的进程所访问，并且对其他namespace下的进程不可见。

At first, it sounds useful, since it allows to sandbox each container within its own directory, hiding other containers.

咋看之下，这样好像挺好用，因为这样可以把每个容器自己的目录结构做成一个沙盒，和其他容器隔离开来。

At a second glance, is it really that useful? After all, if each container is chroot‘ed in a different directory, container C1 won’t be able to access or see the filesystem of container C2, right? Well, that’s right, but there are side effects.

但仔细想想，真的有那么好用么？毕竟，如果把每个容器放在不同的目录下，然后分别使用chroot，这样容器C1就没法看到或访问容器C2的文件目录了，对吧？是的，没错，但这样做会有一些副作用：

Inspecting /proc/mounts in a container will show the mountpoints of all containers. Also, those mountpoints will be relative to the original namespace, which can give some hints about the layout of your system – and maybe confuse some applications which would rely on the paths in /proc/mounts.

在一个容器下查看/proc/mounts会显示所有容器的mountpoints。而且，这些mountpoints的路径会和顶层的namespace相关，这可能会泄露出一些关于你的系统结构的线索，并且可能会让一些依赖/proc/mounts内路径的程序感到迷惑。

######**译者注:此处的容器是指用chroot形成的沙箱环境。

The mnt namespace makes the situation much cleaner, allowing each container to have its own mountpoints, and see only those mountpoints, with their path correctly translated to the actual root of the namespace.

使用mnt namespace能允许每个容器拥有自己的mountpoints，并且在/proc/mounts里只包含这些已经将路径转换为当前namespace根目录的mountpoints。

### The uts namespace

Finally, the uts namespace deals with one little detail: the hostname that will be “seen” by a group of processes.

最后，uts namespace处理一个小细节：容器内的进程所看到的hostname。


Each uts namespace will hold a different hostname, and changing the hostname (through the sethostname system call) will only change it for processes running in the same namespace.

每个uts namespace会有一个不同的hostname，改变hostname（通过sethostname系统命令）只会影响到同一个namespace中的进程。

######**译者注：uts namespace还处理每个容器看到的domainname，这篇文章作者写于12年11月，Linux3.8是13年2月发布的，实现了一个新的user namespace，可参考：http://lwn.net/Articles/531114/

## Creating namespaces

Namespace creation is achieved with the clone system call. This system call supports a number of flags, allowing to specify “I want the new process to run within its own pid, net, ipc, mnt, and uts namespaces”. When creating a container, this is exactly what happens: a new process, with new namespaces, is created; its network interfaces (including the special pair of interfaces to talk with the outside world) are configured; and it executes an init-like process.

通过clone这个系统调用可以创建namespace。这个系统调用支持许多标志，让用户可以指定：“我希望新进程可以使用自己的pid，net，ipc，mnt和uts namespace”。创建一个容器的过程是这样的：首先创建一个带有许多新的namespace的进程；然后配置它网络接口（包括那个用于连接外部网络的特殊接口对）；最后执行一个类似init的进程。

When the last process within a namespace exits, the associated resources (IPC, network interfaces…) are automatically reclaimed. If, for some reason, you want those resources to survive after the termination of the last process of the namespace, there is a way. Each namespace is materialized by a special file in /proc/$PID/ns. Using mount --bind on one of those special files, each namespace can be retained for future use.

当一个namespace中最后一个进程退出之后，相关的资源（IPC，网络接口等等……）会自动被回收。如果因为某些原因，你不希望在这时回收资源的话，是有一个方法。namespace中的每个进程都有一个/proc/$PID/ns目录，里面的每个文件对应一个namespace类型。用mount --bind来挂载其中的某个文件，这样，对应的namespace就可以被保留，即使其中的所有进程都退出了。

######**译者注：这里译者根据http://lwn.net/Articles/531381/，对原文的意思稍加扩充。

Each namespace? Not quite. Up to (and including) kernel 3.4, only ipc, net, and uts namespaces appear here; mnt and pid namespaces do not. This can be a problem in some specific cases, as we will see in the next paragraph.

######**译者注：以上这段过时了，说的是Linux3.4内核下只看得到ipc，net，和uts namespace，在Linux3.8中已经修复了。具体参见：http://lwn.net/Articles/531381/

## Attaching to existing namespaces

It is also possible to “enter” a namespace, by attaching a process to an existing namespace. Why would one want to do that? Generally, to run an arbitrary command within the namespace. Here are a few examples:

通过把一个进程附加（attach）到到一个现有的namespace，可以让我们“进入”到一个namespace。为什么会想要这么做呢？一般情况，是为了在namespace中运行某个命令。下面是一些例子：

- you want to setup network interfaces “from outside”, without relying on scripts inside the container;
- you want to run an arbitrary command to retrieve some information about the container: you could generally obtain the same information by peeking at the container “from outside”, but sometimes, it might require specially patched tools (e.g. if you want to execute netstat);
- you want to obtain a shell within the container.

- 你想要从容器外设置容器的网络接口，而又不想依赖容器内的脚本。
- 你想要运行某个命令来得到容器的一些信息：一般情况下，你可以从外部查询容器而得到相同的信息，但有时，你还是会需要一些特殊的工具（比如需要查看netstat的信息）。
- 你想要在容器内获得一个shell

Attaching a process to existing namespaces requires two things:

把进程附加到现有的namespace需要这样两件事：

- the setns system call (which exists only since kernel 3.0, or with patches for older kernels);
- the namespace must appear in /proc/$PID/ns.

- setns 系统调用（只在内核3.0版本之后才有，老版本内核需要补丁）；
- namespace需要出现在/proc/$PID/ns

Wait, we mentioned those special files in the previous paragraph, and we told that only ipc, net, and uts namespaces were listed under /proc/$PID/ns! So how can we attach to existing mnt and pid namespaces? We can’t – unless we use a patched kernel.

Combining the necessary patches can be fairly tricky, and explaining how to resolve conflicts between AUFS and GRSEC could almost require a blog post by itself. So, if you don’t want to run an overly patched kernel, here are some workarounds.

- You can run sshd in your containers, and pre-authorize a special SSH key to execute your commands. This is one of the easiest solutions, but if sshd crashes, or is stopped (intentionally or by accident), you’re locked out of the container. Also, if you want to squeeze the memory footprint of your containers as much as possible, you might want to get rid of sshd. If the latter is your main concern, you can run a low profile SSH server like dropbear, or you can start the SSH service from inetd or a similar service.
- If you want something simpler than SSH (or if you want something different than SSH, to avoid interferences with sshd custom configurations), you can open a backdoor. An example would be to run socat TCP-LISTEN:222,fork,reuseaddr EXEC:/bin/bash,stderr from init in your containers (but make sure that port 222/tcp is correctly firewalled then).
- An even better solution is to embed this “control channel” within your init process. Before changing its root directory, the init process could setup a UNIX socket on a path located outside the container root directory. When it will change its root directory, it will retain its open file descriptors – and therefore, the control socket.

######**译者注：上面这些讲的是/proc/$PID/ns中没有mnt和pid namespace的对应文件，用怎样的变通方法来解决问题。在Linux3.8中已经不存在这个问题。

## How dotCloud uses namespaces
## dotCloud怎样使用namespaces

In its early age, the dotCloud platform used plain LXC (Linux Containers), and therefore, made implicit use of namespaces.

dotCloud平台的早期，我们使用纯LXC（Linux Containers），因此，隐式地使用namespaces。

Very early, we deployed patched kernels, allowing us to attach arbitrary processes into existing namespaces – because we found it to be the most convenient and reliable way to deploy, control, and orchestrate containers. The platform evolved, and while the original “containers” are being stripped down (and bearing less and less similarity with usual Linux Containers), we still use namespaces to isolate applications from each other.

更早的时候，我们部署打补丁的内核，这允许我们把任意进程依附到现有的namespace上——因为我们发现这是部署，控制和协调容器最方便，且最可靠的方法。在平台升级，和最早的容器被越来越多的定制化之后（变得越来越不像通常的Linux Containers），我们仍然使用namespace来隔离不同的应用。

######**这段好像有点诡异。。
=======
>>>>>>> parent of 21a8ec4... First version:PaaS under the hood, episode 1: kernel namespaces.md
