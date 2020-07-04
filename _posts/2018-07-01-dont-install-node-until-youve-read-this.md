---
layout: post
title:  "Don't Install Node Until You've Read This (Or, How to Run Node the Docker Way)"
date:   2018-07-01 13:30:01 +0000
comments: true
tags: 
 - node
 - docker
---

We need Node for some application or other--perhaps we're creating a microservice or just want to follow along with a tutorial.

But most places you start with suggest that the first step is to install Node for your operating system. Perhaps you're on a Mac so now you have to start thinking about whether you should also install Homebrew or MacPorts.

Or you're on Ubuntu so you head in the `apt-get` direction...except before you know it, to get the latest version you find yourself using `curl` to pipe some script to your shell.

Windows? You could just use the Windows installer but as with macOS you ponder whether it's time to embrace the Chocalatey or Scoop package managers.

In this blog post we'll look at how skipping over all of this and heading straight to a Docker environment makes it much easier to manage your Node applications and development workflow, and whats more gets you going with best practices right from the start.

<!--snip--->

# Docker First

![Docker Logo](/images/uploads/docker.png)

Whichever route we go with installing Node the OS-specific way, we now have two problems; the first is that the way that we install Node is different on each platform, and damn, that's annoying. And number two we now have Node installed *globally* on our laptop. Why so sad? Well now if we want to use different versions of Node for different projects we have to faff about with something like `nvm`. (And if you were planning on running a Python project it's the same story, with `virtualenv`.)

So do yourself a favour and [get Docker installed](https://docs.docker.com/install/). True, how you install Docker will also be different for different platforms--Ubuntu is slightly different to Mac and Windows. But this initial effort will repay you later because now you'll have a *standard* way to install Node, Ruby, Python, TensorFlow, R...whatever language you're using for your projects--or perhaps more likely nowadays, *languages*--just got way easier to manage.

So assuming that you now have Docker, let's get a development environment set up so that you can get back to that tutorial or project.

# Running Node

![Node Logo](/images/uploads/node.png)

First, create a new directory for your project:

```shell
mkdir new-project && cd new-project
```

and then launch the latest version of Node:

```shell
docker run -it --rm node:10.5.0-alpine
```

If you haven't run this version of Node before, Docker will download it for you. After a bit of toing and froing you'll be left with the usual Node command prompt. Type something like `5+6` and press return to check all is well, and then press `[CTRL]+D` to exit.

If you are reading this in the future you might want to find out what the most recent version number is; just head to the [Docker Hub page for the official Node Docker image](https://hub.docker.com/_/node/).

## Interactive Containers

We executed the `docker run` command with a couple of options. The first--the `-it` part--is a combination of the two options, `-i` and `-t`. It's these options together that mean we can interact with the running container as if it was our normal shell, accepting input from our keyboard and sending output to our display.

## Disposable Containers

The `--rm` option causes the container to be deleted when we exit. It's a good habit to get into to delete containers as we go along, because it gets us into the mindset that our containers are *disposable*. This is particularly important when it comes to deployment because we don't want our container to hold any state internally--any updates or processing should result in writes to external services such as a connected file system, cloud storage, queues, and so on. By taking this approach it's really easy to upgrade our images to newer versions when necessary--we just throw away the old ones and launch completely new ones.

(It will also make it easier to scale, since we can just launch a bunch more containers when we need to do more work, and provided that all state is maintained *outside* of the containers this becomes straightforward.)

### Bonus Points: No SSH

If you really want to get into good habits with your Docker containers then also avoid the tempation to SSH into a running container to see what's going on. There's nothing worse than making a tweak to fix something, logging out, and then forgetting what was changed. The service may now be running again and your boss thinks your are flavour of the month, but it's fragile. Deploy again and you overwrite those changes. Far better to fix the problem in your deployment scripts, then simply tear down the faulty service and launch another. The changes are now clear to see in source control and reproducible.

## Versions

Beyond the command-line options to `docker run`, there are also a few things to note about the Node Docker image that we've used (the `node:10.5.0-alpine` part).

First, it's worth being specific about the version number of Node that you are using, since it makes it easier to force updates and to know what is being deployed. If we were to only specify 'version 10':

```shell
docker run -it --rm node:10-alpine
```

or even 'the latest version of node':

```shell
docker run -it --rm node:alpine
```

then although on the first time through we'll get `10.5.0`, once the images are updated at some later point, we won't pick up the same version on subsequent runs. At some point using `node:10-alpine` in the command will cause us to pick up version `10.6.0` or `10.7.0` of Node. And using `node:alpine` will at some point cause us to get version `11` and onwards.

However, if we choose a specific version like `10.5.0` then although we also won't get updates automatically, it will be a simple case of updating to `10.5.1` in our build files, when we are ready to force a download of the latest changes.

This is particularly important when it comes to deploying applications later on (or sharing your code with other people), since you want to be able to control what version appears where. And perhaps more to the point, when you are troubleshooting you want to know for sure what version was used.

### Controlled Updates

It's tempting of course to want to 'always use the latest'; after all, the latest will be faster, won't it? And won't it have the latest security patches? This is true of course, but in the quest for building a reliable infrastructure you should aim to *control* updates to the foundations. This means that if you have a bunch of code that is working fine on version `10.5.0`, nicely passing all of its tests and performing well, then a move to another version of Node should be something that is planned and tested. The only *real* pressure to move versions comes with the point releases such as `10.5.1` or `10.5.2`, since they will contain security patches and bug fixes; a move to `10.6` or higher is certainly a 'nice to have', but if your code is working and your service is running, then you will definitely want to consider whether your time is better spent elsewhere.

## Base OS

![Alpine Logo](/images/uploads/alpine-linux.svg)

The second thing to note about the Node Docker image selection, is that we've used the `alpine` version of the image which uses [Alpine Linux](https://alpinelinux.org/about/) as the base operating system. This is the lightest of the Node images, and only provides the bare minimum of an operating system to get Node running--we're most likely creating microservices, after all.

You've probably come across the `alpine` project but if you haven't, take a look; it's being used right across the Docker ecosystem to keep Docker images light.

It should be said as well that 'light' doesn't just mean small for the sake of size--that's all good of course, since it reduces the amount of data flying around your network. But in the case of a deployed service 'light' also means reducing the number of moving parts that can go wrong. If you start with something big like a Ubuntu base image you're bringing in a bunch of unnecessary code and so increasing the possibility of something going wrong that wasn't important in the first place. Imagine some nefarious outsider taking advantage of a security hole in Ubuntu, in a service that you didn't even need!

(You may have come across the expression 'reducing attack surface'; this is *exactly* what is being referred to.)

So keep it small, tight, and controlled...and most of all, *secure*.

### Building Your Own Base Images - Don't!

And it should probably go without saying that you don't want to be building your own base images. The various Docker Node images, for example, are maintained by the Node project itself, so if anyone is going to know how to build a secure, fast and reliable image it's them. What's more, if anything does go wrong there is a whole community of people using the image and reporting issues; you'll invariably find a solution very quickly.

# A Development Environment

So we have chosen a Node image, and we have it running from the command line. Let's press on with our development environment.

In order to be able to update files in our project directory we need to give our Node application 'access' to that directory. This is achieved with the 'volume' option on the Docker command. Try this:

```shell
docker run -it --rm -v ${PWD}:/usr/src/app node:10.5.0-alpine \
  /bin/sh -c "touch /usr/src/app/README.md"
```

This will:

* create a directory *inside* your Docker container (at `/usr/src/app`), and make it refer to your current working directory *outside* your container (the `${PWD}` part);
* launch the Bash shell (rather than Node), to run the `touch` command which will create a `README` file.

The command should exit cleanly. Check your current directory to ensure that the file has been created:

```shell
$ ls -al
total 0
drwxr-xr-x   4 markbirbeck  staff  136  1 Jul 13:26 .
drwxr-xr-x  10 markbirbeck  staff  340  1 Jul 11:47 ..
-rw-r--r--   1 markbirbeck  staff    0  1 Jul 12:58 README.md
```

This is a laborious way to create a file, but we just wanted to check that our Docker container was able to 'see' our laptop project directory and that it could update files within it.

We now have *two* ways that we can work on our project: we can either fire up `vi` from *inside* the container and make edits which will immediately be mirrored to our working directory on our laptop; or we can use our familiar laptop tools--like Visual Studio Code, Sublime Text, and so on--to create and edit files *outside* the container, knowing that changes will be immediately mirrored to the `/usr/src/app` directory within the container.

Either way, we can now develop in pretty much the same way as we normally would on our laptop, but with an easy to manage Node environment, courtesy of Docker.

# Opening Ports

One last thing. Let's say we got started with Node by following [the little intro on the Node site](https://nodejs.org/en/docs/guides/getting-started-guide/). You'll see that it sets up a 'hello world' web server and suggests that the page can be viewed at `http://localhost:3000`. Go ahead and create that `app.js` file in your current directory...but there's no point in running it since as things stand with our *Docker* development environment approach, this server won't work.

However, just as we saw earlier that we can map directories between the host and the container we can also map ports. The first step is to add the `-p` option to our command like this:

```shell
docker run -it --rm -v ${PWD}:/usr/src/app -p 3000:3000 node:10.5.0-alpine \
  /bin/sh
```

We can now access port 3000 *inside* the container by making requests to port 3000 on our host machine, which satisfies the `http://localhost:3000` part of the Node tutorial.

But there is one last minor tweak we'll need to make; when the server launches it will listen on the IP address `127.0.0.1` which would be fine on our laptop, but is no good inside a container. We might use this address to prevent our server being reached from outside of our laptop, but in the case of a Docker container there is a network connection from our laptop to the container (think of them as separate machines), so keeping things 'private' *inside* the container will just mean that nothing is reachable.

All we need to do is change the file that was provided on the Node site, and modify the `hostname` variable from `127.0.0.1` to `0.0.0.0`. This will tell the server to listen to *all* IP addresses within the container, not just `localhost`. We can still ensure that our server is not reachable from outside of our laptop if we want, by modifying the Docker command to this:

```shell
docker run -it --rm -v ${PWD}:/usr/src/app -p 127.0.0.1:3000:3000 \
  node:10.5.0-alpine /bin/sh
```

I.e., the mapping from host port to container port should only take place on `127.0.0.1` rather than on `0.0.0.0` (which is the default for a port mapping).

Whether you modify the port setting when you run the command or not, once the `app.js` file has this minor change then the server can be launched from inside the container. Change directory to where the `app.js` file is, and then launch it:

```shell
cd /usr/src/app
node app.js
```

Now you should be able to reach the 'hello world' page from the host machine by visiting `http://localhost:3000`.

# Next Steps

Assuming all is well, we can now carry on with whatever project or tutorial we were following. Anywhere that the tutorial tells us to run something from the command-line we make sure to do it from *inside* the container by firing up the Bash shell. If the project requires that we expose a different port then just change the `-p` option (or add more mappings if necessary).

There are a lot more ways that we can improve our develoment environment; we can:

* [bring in Docker Compose to shorten our command lines](/2018/07/02/your-development-workflow-just-got-better-with-docker-compose.html);
* add more directory mappings so that modules installed with `npm install` stay *inside* our container;
* create test containers that include runners like Mocha or TAP;
* launch local Nginx servers that will mirror our live deployments.

But all of these will build on the basic setup we have here. We'll drill into these techniques in future posts.

Good luck with your projects!