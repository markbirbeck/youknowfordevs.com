---
layout: post
title:  "Your Development Workflow Just Got Better, With Docker Compose"
date:   2018-07-02 13:30:01 +0000
comments: true
tags: 
 - docker
 - docker compose
 - node
---

In a previous post we saw how to [set up our basic Node development environment using Docker](/2018/07/01/dont-install-node-until-youve-read-this.html). Our next step is reduce the size of these unwieldy `docker run` commands. This is not just because of their unwieldiness but also because if we just type them from the command-line then we don't have an easy way to share what we're doing--not just with other people but with ourselves, tomorrow, when we've inevitably forgotten what we were doing today!

<!--snip-->

So before we forget the command we were running in the previous post, let's lock it down in a file that we can use repeatedly.

But in what file, you ask?

# Docker Compose

The tool we're going to use to capture these kinds of commands is Docker Compose. This app will have been installed for you when you installed Docker (assuming that you took the advice of our previous post to [embrace Docker](/2018/07/01/dont-install-node-until-youve-read-this.html#docker-first)). Docker Compose is an *incredibly* handy utility because it allows us to use a YAML file to create definitions for Docker commands, rather than having to use command-line options. This means we can easily share and version our commands.

The YAML file can also be used to manage a group of containers that we want to launch at the same time--perhaps our microservice needs a MySQL database or a RabbitMQ queue--and as if that wasn't enough, the same file format can also be used to describe a Docker swarm stack--a collection of services that will all run together--when it comes time to deploy our application.

Just as in the previous post we suggested that applications should no longer be installed locally but instead run inside Docker containers, now we want to just as strongly argue that no activity that can be performed in the creation of your application--whether linting, testing, packaging, deploying--should be carried out without it being captured in a Docker Compose file.

But before we get too excited, let's go back to the command we were running in the earlier post (which launches a development container in which we run Node) and let's convert it to use Docker Compose.

## A Docker Compose Configuration File

Recall that the command we were running was:

```shell
docker run -it --rm -v ${PWD}:/usr/src/app -p 127.0.0.1:3000:3000 \
  node:10.5.0-alpine /bin/sh
```

To turn this into a Docker Compose file, fire up your favourite editor and create a file called  `docker-compose.yml` into which you've placed the following:

```yaml
version: "3.2"

services:
  dev:
    image: node:10.5.0-alpine
    ports:
    - "127.0.0.1:3000:3000"
    volumes:
    - .:/usr/src/app
    command: ["/bin/sh"]
```

You can probably figure out which parts of the original command-line map to which entries in this Compose file, so we'll just flag up a couple of things that might not be immediately obvious.

First, the entry `dev` is just the name of our *service*. It can be anything we like, and there can be more than one of these entries in a file. We'll see in a moment how it's used to indicate what we want to launch.

(A service is the term Docker Compose uses to describe running containers. The reason it doesn't use the term *container* in the way that we would if we were using the `docker run` command is that a service has extra features such as being able to comprise more than one instance of a container.)

Next you probably noticed that the port mapping now has quotes around it; on the command line we had `-p 127.0.0.1:3000:3000` whilst in the compose file we have `"127.0.0.1:3000:3000"`. This is a best practice due to the way that YAML files are processed. If a port lower than 60 is mapped and no IP address is specified (for example, `40:40`) then the parser will not treat it as `40` followed by `40`, but as a base 60 number. You *could* just remember that you need quotes when using ports below 60, but most Docker Compose files you'll see will have quotes placed around *any* port number, which is a little easier to remember.

Finally, you will also have spotted that the `${PWD}` part of our `docker run` command has now been replaced with `.`, i.e., the current directory. Docker Compose doesn't need the environment variable when mapping volumes, which makes things a bit easier. Paths in the YAML file are always relative to the file itself (and relative paths are supported).

## Launching Our Development Container

Now we have our configuration set up it's a simple matter of running the Docker Compose command with the name of our service. Run the following command and you should have launched the development environment again:

```shell
docker-compose run --rm --service-ports dev 
```

Ok...so it's still not the shortest command on the block--we'll see in a future post how we can get this down further. But it's a lot easier to remember than the long `docker run` command we had before. And what's more, it will *always be the same* no matter what changes you make to the configuration file; any additional options we want to add to our `docker run` will go in our Docker Compose file, clearly documented and under source control.

Just to wrap up this section, we'll quickly explain the parameters that we need to pass to `docker-compose run`. The first is `--rm` which is exactly the same as the option we were using with `docker run`--when the command has finished running our container will be deleted.

The second is `--service-ports` which instructs Docker Compose to make available any port mappings we define in the Compose file. It's a little annoying to have to add this parameter, and you'll find many discussion threads that argue that this behaviour should be the default. But the logic is fair; if we are launching a number of connected services, such as a web server and a MySQL database, we don't necessarily want every single port to be mapped to our host machine. In the example of a web server and MySQL server for example, there is no need to expose MySQL's port `3306` on our laptop since it is only needed by the web server connection to the backend. Docker Compose will create a network that the web server and MySQL can use to communicate with each other.

So there we have it; run that command, and we will get a shell prompt, and then we can launch our web server in exactly the same way as we did in the previous post, when using `docker run`:

```shell
cd /usr/src/app
node app.js
```

## Working Directory

We said a moment ago that one of the advantages of using Docker Compose is that we can add additional options without changing the way we run the command. An example would be to get Docker to change to the working directory for us, i.e, to remove the need for the `cd /usr/src/app` step in our sequence, above.

To do this we only need to add the `working_dir` option to the YAML file:

```yaml
version: "3.2"

services:
  dev:
    image: node:10.5.0-alpine
    working_dir: /usr/src/app
    ports:
    - "3000:3000"
    volumes:
    - .:/usr/src/app
    command: ["/bin/sh"]
```

And to stress again, we still launch our development environment in exactly the same way as we did before--the only changes are to the configuration file:

```shell
docker-compose run --rm --service-ports dev 
```

This time our command-line prompt will have us sitting in the correct directory, and we can launch the server directly:

```shell
node app.js
```

## Changing Launch Commands

But we can go a bit further here; we'll rarely need to be 'inside' the container doing stuff, since we'll be using our favourite editor running on our laptop (remember [we've mapped our project directory into the container so that our laptop and the container both have access to our files](/2018/07/01/dont-install-node-until-youve-read-this.html#a-development-environment)). So we'll probably find ourselves more often than not invoking our container and then running the server. So we could change the command that is run inside the container from one that launches a Bash shell to one that launches the server:

```yaml
version: "3.2"

services:
  dev:
    image: node:10.5.0-alpine
    working_dir: /usr/src/app
    ports:
    - "3000:3000"
    volumes:
    - .:/usr/src/app
    command: ["/bin/sh", "-c", "node app.js"]
```

### Making a Clean Exit

You probably spotted that the command we added was not what we might have expected:

```yaml
    command: ["node", "app.js"]
```

but:

```yaml
    command: ["/bin/sh", "-c", "node app.js"]
```

The background as to why is that if we use the first version of the command which simply runs `node` with `app.js` as the parameter, then when we try to exit the server with `[CTRL]+C` nothing will happen and we'll have to find some other way to kill the server. This is because the Node app does not process a `SIGTERM` signal (a `[CTRL]+C`) correctly when Node is running as the primary, top-level application in a container (what you'll often see described as *running as PID 1*).

However the Bash shell *does* handle the whole `SIGTERM` dance correctly, and will cleanly shut down our server when it receives `[CTRL]+C`. So all we need to do is run our server inside a shell.

If you need (or want) to understand this in more detail then [search online for something along the lines of "pid 1 docker node"](https://www.google.co.uk/search?q=pid+1+docker+node) and you'll find a number of articles. If you just want to cut to the chase then read the section [Handling Kernel Signals](https://github.com/nodejs/docker-node/blob/master/docs/BestPractices.md#handling-kernel-signals) in the [best practises guidance for using Node in Docker](https://github.com/nodejs/docker-node/blob/master/docs/BestPractices.md).

## Multiple Services

Of course, if we think we might need *both* of these commands--the one to launch a Bash shell inside the container, ready for playing around, and the one to launch the server--then instead of overwriting our first, we can just add a second entry to our Docker Compose file:

```yaml
version: "3.2"

services:
  shell:
    image: node:10.5.0-alpine
    working_dir: /usr/src/app
    ports:
    - "3000:3000"
    volumes:
    - .:/usr/src/app
    command: ["/bin/sh"]

  serve:
    image: node:10.5.0-alpine
    working_dir: /usr/src/app
    ports:
    - "3000:3000"
    volumes:
    - .:/usr/src/app
    command: ["/bin/sh", "-c", "node app.js"]
```

We've changed the name of the shell version from `dev` to `shell` to indicate what it's used for, which means we can now launch the server with:

```shell
docker-compose run --rm --service-ports serve
```

### Don't Repeat Yourself

One last tip involves a way to reuse the common settings we have in our file. As you can see the only difference between our two services is in the `command` value. Ideally we'd like to place all of the other values into some common collection and share them between both services.

This is possible in [version 3.4 onwards of the Docker Compose file format](https://docs.docker.com/compose/compose-file/) by using YAML anchors:

```yaml
version: "3.4"
x-default-service-settings:
  &default-service-settings
    image: node:10.5.0-alpine
    working_dir: /usr/src/app
    ports:
    - "3000:3000"
    volumes:
    - .:/usr/src/app

services:
  shell:
    << : *default-service-settings
    command: ["/bin/sh"]

  serve:
    << : *default-service-settings
    command: ["/bin/sh", "-c", "node app.js"]
```

So note first that the `version` value has been updated at the top of the document. Then, any block that we want to create for sharing goes at the top level with an `x-` prefix--that's how we tell Docker Compose not to process this block as some configuration.

Within the custom block we set an anchor (the `&default-service-settings` part) and give it any name we want. Then finally we can refer to that block by referencing the anchor with the `<<` syntax.

# Next Steps

We've taken our [original `docker run` command](/2018/07/01/dont-install-node-until-youve-read-this.html) and converted it to use Docker Compose, making complex configurations much easier to manage. We've also added some additional commands to help with our development process. And we also now have a way to keep a collection of commands under source control. We can now build on this approach to:

* add more directory mappings so that modules installed with `npm install` stay *inside* our container;
* add entries for test containers that include runners like Mocha or TAP;
* add entries for commands that help the build process, for example using Webpack or Parcel;
* launch local Nginx servers that will mirror our live deployments.

We'll drill into these techniques and more in future posts.

Good luck with your projects!
