#Docker template for building CyanogenMod

#编译 CyanogenMod 的 Docker 模板

***

Building [CyanogenMod](http://www.cyanogenmod.org) requires quite a lot of work. You will need to install a large number of dependencies, and you will need to read through lots of documentation.
[Docker](http://docker.io) is a rather new software to automate the deployment of applications inside a software container.

编译 [CyanogenMod](http://www.cyanogenmod.org) 需要很多的工作。你需要安装大量的依赖包，你还需要阅读很多文档。[Docker](http://docker.io) 是一个在容器自动内部署应用的软件。

***

Here is a Docker container for running an environment which contains everything that is needed to compile CyanogenMod. It will be very easy to install, and it will just work! The Github page contains some further information on how to get started.

这个 Docker 容器已经包含了编译 CyanogenMod 所依赖的所有环境。 它非常容易安装，装好即用。 Github 项目页面有更多如何使用安装使用的信息。

***

NOTE: You will need to install Docker to proceed: [https://www.docker.io/gettingstarted/](https://www.docker.io/gettingstarted/)

注意：你需要根据 Docker 的文档先安装 Docker ：[https://www.docker.io/gettingstarted/](https://www.docker.io/gettingstarted/)

***

How to build:

编译：

***

```
git clone https://github.com/stucki/docker-cyanogenmod.git
cd docker-cyanogenmod
./build.sh
```

How to run:

运行：

***

```
cd docker-cyanogenmod
./run.sh
```

***
How to build CyanogenMod for your device:

为你的设备编译 CyanogenMode：

***

```
repo init -u git://github.com/CyanogenMod/android.git -b cm-11.0
repo sync
vendor/cm/get-prebuilts
source build/envsetup.sh
breakfast <device codename>   # example: breakfast grouper
brunch <device codename>      # example: brunch grouper
```

***

Download:

下载：

Github URL: https://github.com/stucki/docker-cyanogenmod

Github 库地址：[https://github.com/stucki/docker-cyanogenmod](https://github.com/stucki/docker-cyanogenmod)

***

ChangeLog:

***

```
2014-02-20

* Add note about running get-prebuilts

2014-02-16

* Initial release
```

***

Any feedback is welcome. Enjoy!

欢迎提交反馈。
