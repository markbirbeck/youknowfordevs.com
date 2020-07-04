---
layout: post
title:  "Disposable Laptops With Docker Compose And NPM"
date:   2017-07-23 15:11:01 +0000
comments: true
tags: 
 - docker
 - docker compose
---
Switching laptops a lot more frequently than I would want has led over the years to my trying a variety of ways to keep track of the applications that I need to install to get up and running quickly. In this post I'll look at where I've got to with this.

<!--snip-->

_[If you want to look at the results of this exploration before you look at the 'why' then head over to [docker-compose-run](https://github.com/markbirbeck/docker-compose-run) to see how to run cross-platform apps. Also, look at [https://github.com/markbirbeck/mydesktop](https://github.com/markbirbeck/mydesktop) for an example of how to quickly install a collection of these cross-platform apps on any device.]_

Just when you think you've got your laptop set up just how you like it, something happens that means you have to start again. Perhaps you upgraded your hard-drive to an SSD and had to reinstall everything. Or maybe your fan broke and you had to switch to another laptop whilst it was repaired. (Only to have the wireless network card break on the stand-in, requiring _another_ swtich.)

Or maybe the laptop didn't break at all, and you left it on an underground train after a cookery class--no doubt distracted by having too many bags of food to carry. The upside of this particular event is that losing an entire computer means you finally have a perfect excuse to buy Linux rather than Mac, as you've long been telling yourself you would.

All of these changes has led over the years to a variety of ways to document and track how a machine should get set up. My current interest is laptops but in the past I needed to address the same challenge with servers. From [simple shell scripts in RightScale](http://markbirbeck.com/2008/02/27/sugarcrm-rightscale-and-ec2/) back in 2008, through [Chef (and VirtualBox)](http://markbirbeck.com/2012/03/16/using-knife-to-launch-ec2-instances-without-a-chef-server/) in 2012, and finally to Docker today, I've used a myriad of tools to get servers deployed and running as quickly as possible (and to record exactly how I did it).

The general logic with servers has been to start with a clean slate, run some scripts to configure things how you'd like, and then let it go. Crucial to the approach is to not assume that the server will last forever, which means nothing gets placed on the machine that can't be 'recreated' through the install scripts.

But for laptops it hasn't been so straightforward. For a start, there isn't an easy concept of 'run these scripts to get a standard laptop' because laptops tend to change a lot during normal use. What I've usually done--and I've seen many others do the same--is create a script that installs all the software I like _on a new machine_ and then try to keep that script up-to-date ready for the fateful day when it will be used again.

One obvious problem with this is that you have to remember to update the script when you install new software...and then update it again when you remove software. Another issue is that the only time you ever run this script is when you have a brand new laptop, so you can't be sure that it will always work (or conversely, the only time it will fail is when it's crucial that it doesn't). And yet another problem is that many applications don't have easy installations, so your scripts end up not being a collection of actions but a bunch of comments that remind you of where to download a zip file from, or which special incantation needs to be run to apply a patch to a file to fix a buggy release.

But perhaps the most annoying issue is that my ['Mac From Scratch' script](https://gist.github.com/markbirbeck/2fb12460eb69484a87b3) ends up being very different to my ['Ubuntu From Scratch' script](https://gist.github.com/markbirbeck/fef06d10cc90e7000fad39beb7216701)--partly because there is different software on the different platforms, but also because, even when both platforms use the same software they each have different package managers.

Why would I need two sets of scripts? Well at the moment I have three old Mac's lying around plus the Ubuntu laptop I'm writing this post on, and since I can't afford downtime, in an emergency I'd like to be able to switch any of those machines into service as quickly and easily as possible.

So lately--and with this goal in mind--I've been taking a different approach, made possible by first Docker, and then Docker Compose...all bound together with NPM.

# Dockerising The Desktop

The Docker integration on the Mac (and Windows, for that matter) has come on in leaps and bounds in the last year or so, so an obvious question now is why not run our desktop apps through Docker? [^2]

This has a number of advantages. The most obvious is that suddenly you're running exactly the same apps on _any_ of your desktops. For example, you might use [bitlbee](https://www.bitlbee.org/main.php/news.r.html) to interact with all of your online chat services. Get this set up in Docker and you can run the exact same configuration on any machine you own, regardless of whether it's Linux, OS X or Windows. More than that, as you tweak and poke and prod your configuration you can keep track of the changes in a Dockerfile in GitHub--or whatever version control system you prefer--rather than adding and removing comments in a Gist.

If you need more advantages, perhaps my favourite is that _containerising your desktop_ keeps down the clutter that can happen when installing lots of dependencies. For example: I've never really liked Ruby, I'm afraid [^1], so I'd really rather not have loads of gems cluttering up my environment.  Of course, if I'm running an application that is written in Ruby then I'll need those gems, but if I move away from the app to use another I don't want the app (and Ruby) to put up a fight when I come to uninstall it. By keeping all of the gems inside a container they're all gone in one fell swoop.

And the icing on this already delicious cake is that if you decide to move an app to the cloud--and something like _bitlbee_ is a prime candidate for that--then now that you have a `Dockerfile` it's just a question of deployment.

# Let's Jekyll

To illustrate all of these ideas, let's get a blog set up using Jekyll. 

## Docker

It so happens that the Jekyll development team have created a number of Docker images that contain everything we need to get going. The images are hosted on Dockerhub at [jekyll/jekyll](https://hub.docker.com/r/jekyll/jekyll/).

The general pattern with apps that are running in a Docker container is to provide a load of preamble and then the specific command that we want to run. The 'specific command' part that gets Jekyll to take the files in the current directory and build our blog looks like this:

```shell
jekyll build
```

With Docker we add the preamble at the front, so it will look something like this:

```shell
[preamble] jekyll build
```

The preamble will get Docker to launch the correct container before running the command that we add to the end--i.e., the `jekyll build` part-- _inside_ the container. We'll look later at how we can refine this preamble, but for now let's spell it out.

The [Jekyll Docker image usage wiki page](https://github.com/jekyll/docker/wiki/Usage:-Running) advises the following to run a command:

```shell
docker run \
  --rm \
  --label=jekyll \
  --volume=$(pwd):/srv/jekyll \
  -it \
  -p 127.0.0.1:4000:4000 \
  jekyll/jekyll \
  jekyll serve
```

This pattern is pretty much the same as we'll use for all of our commands so it's worth breaking apart:

### Removing Containers On Close (`--rm`)

We tell Docker to remove the container after it has finished running because there is nothing 'inside' the container that we want to keep after it has exited, and we don't want to have to periodically delete these finished containers to save space.

### Labelling The Container (`--label`)

We give the container a label of `jekyll` to make it easier to keep track of.

### Providing Access To Our Local Files (`--volume`)

We need to give the container access to the blog files that we have in the current directory. We don't want to put the files _inside_ the container, because then we'd have to work out how to share the container with our other computers as well as having to deal with its lifecycle. It's much easier to treat the Docker container itself as disposable and then worry about giving it access to our data which we'll store elsewhere. To share the blog posts on our different computers we can easily mirror our simple markdown files with Dropbox, Google Drive, or whatever.

The pattern we'll see as we use Docker for more and more local apps is that some directory on our local machine--in this case the current working directory, obtained with `$(pwd)`--is made available _inside_ the container, as if it was the directory `/srv/jekyll`.

### Connecting To Our Console (`-it`)

We also need to tell Docker that we want to interact with the session whilst it's running, which we can do using the `-i` and `-t` options. With Jekyll we're mainly interested in any output messages, but there will be other apps--like email and chat clients--where we'll actually interact fully with the running container.

### Mapping The Ports (`-p`)

To get access to the running web server from a web browser on our local machine we need to give the `-p` option. This maps a port _inside_ the container, to a port on our local machine.

## Running the Command

So what happens when we run this command?

Well first, if you haven't run it before, the Docker image `jekyll/jekyll` will be downloaded by Docker, and cached. Once the image has been downloaded it will be used to launch a container with all of the voumes and port mappings that we've given. And once all this is set up, our command (in this case `jekyll serve`) will be run. We'll see Jekyll's output messages in our console and then we can navigate to `http://localhost:4000/` in a web browser.

The command we've just run is a bit of a mouthful so it might be tempting to create a shell script or alias to keep it shorter. However, there is another technique which is part of the Docker ecosystem, and gives us a number of advantages.

We'll look at the benefits in a moment, but for now let's look at how the approach works.

## Docker Compose

The Docker ecosystem comes with a powerful tool called Docker Compose that you can think of as sitting 'above' Docker itself, since it's capable of launching multiple Docker containers as a unit (hence 'compose'). Docker Compose provides a convenient way to specify all of the different aspects of the various containers that you might want to run--volumes, network settings, which other containers to depend on, and so on--which makes it ideal, even when just launching one container.

So let's see what a Docker Compose file that provides the same Jekyll 'preamble' might look like:

```yaml
version: "2"
services:
  jekyll:
    image: jekyll/jekyll
    ports:
    - "4000:4000"
    volumes:
    - ${PWD}:/srv/jekyll
```

 Docker Compose will default to wiring in a terminal for us so we don't need to specify the `-i` and `-t` options we saw before. However, for some reason the default behaviour with `docker-compose run` is _not_ to automatically delete any finished containers, or to map the exposed ports (both of which _do_ happen when using `docker-compose up`), so we'll need to add the `--service-ports` and `--rm` options.

If you create a file called `docker-compose.yml` in your blog directory with the YAML that we have above, you can then get the effect of the longer Docker command by doing this:

```shell
docker-compose run --rm --service-ports jekyll jekyll serve
```

In effect we've replaced the preamble of the long `docker run ...` command with `docker-compose run --rm --service-ports jekyll`, i.e., we've told Docker Compose to launch a Docker container using the options found under the key 'jekyll' in the `docker-compose.yml` file.

## Benefits

We've seen how it works, so now what are the benefits?

An obvious one is that we've reduced the size of the preamble to something a little more manageable; we've taken this:

```shell
docker run \
  --rm \
  --label=jekyll \
  --volume=$(pwd):/srv/jekyll \
  -it \
  -p 127.0.0.1:4000:4000 \
  jekyll/jekyll \
  ...
```

down to this:

```shell
docker-compose run --rm --service-ports jekyll ...
```

But as we said above, we could have done that with an alias or shell script. The big advantage here is that we've captured the detail of the command--the name of the image to download and launch, the volume mapping to use, the ports to expose, and so on--in a platform-independent language that can be checked for validity and easily shared on different machines.

In fact, in the case of this Jekyll example the details of the blogging software you are using could actually be saved as part of your blog; you could save the `docker-compose.yml` file in the same repo as the markdown for the blog, and be up and running on any new machine within minutes, simply by running the `docker-compose` command.

This is not to be taken lightly. It's as if we've installed the blogging software in the same directory as the blog itself, and committed it to source control; we've managed to get all the benefits of linking an app directly to the subdirectory that it acts upon, but with none of the disadvantages.

## Standard Preamble

One last benefit is that by taking a lot of the parameters out into a configuration file we've effectively made our preamble consistent across many apps. This lends itself nicely to a simple alias or shell script that abbreviates things a little further; instead of running this:

```
docker-compose run --rm --service-ports jekyll jekyll serve
```

we could simply do something like this:

```shell
dcr jekyll serve
```

A simple alias to achieve this could be:

```shell
function dcr {
  docker-compose run --rm --service-ports "${1}" "${@}"
}
```

So instead of having one alias for each of our commands, we have a single alias that encapsulates the preamble and can be used for all commands.

## Using A Global `docker-compose.yml` File

Having a local `docker-compose.yml` file in our blog is great, and with this shell function we can certainly get up and running pretty quickly. But what about our other applications? We were talking about how to get our entire laptop up-to-speed as quickly as possible and this doesn't yet do that.

One way we could take this further would be to have a single central Docker Compose file that contains instructions for all of our applications. For example, we might have entries for Jekyll and Mutt:

```yaml
version: "2"
services:
  mutt:
    image: fstab/mutt
    volumes:
    - ~/.mutt:/home/mutt

  jekyll:
    image: jekyll/jekyll
    ports:
    - 4000:4000
    volumes:
    - ${PWD}:/srv/jekyll
```

With a file like this it would then just be a question of placing it somewhere convenient and ensuring that it's mirrored to our Git repo, before modifying the alias so that it finds this file. The `-f` option does this, like so:

```shell
function dcr {
  docker-compose --rm --service-ports -f ~/.dcr/docker-compose.yml run "${1}" "${@}"
}
```

This is a little cumbersome to maintain, and has the disadvantage of being tied to a particular operating system--although it would no doubt be pretty simple to convert.

But maybe we can go one better, by using the Node package manager tools to manage our apps?

## Using NPM Modules

NPM gives us a couple of big benefits that can take our Docker Compose approach to another level. The first is that by treating our apps as a module we can easily keep track of the corresponding `docker-compose.yml` files. 

The second advantage is that NPM has the ability to create symbolic links to applications when a module is installed; we can now get the same functionality that we'd get from aliases and functions, but in a way that will work across operating systems.

So to install our Jekyll Docker shortcut we might simply have to do this:

```shell
$ npm install -g dcr-jekyll
```

We now have a cross-platform way to manage installations of our cross-platform applications!

I've implemented this NPM approach at [docker-compose-run](https://github.com/markbirbeck/docker-compose-run) on GitHub. This module provides the equivalent of the alias to run Docker Compose with the right parameters. It's then used to create application shortcuts such as [dcr-jekyll](https://github.com/markbirbeck/dcr-jekyll) and [dcr-mutt](https://github.com/markbirbeck/dcr-mutt). For a list of applications available see [the wiki page](https://github.com/markbirbeck/docker-compose-run/wiki).

## Reproducible Laptop

So now we're able to run our apps, and we're also able to use the same app across multiple platforms; the final part of the jigsaw is to keep track of the apps that we want to use on any of our devices. And here there is an interesting twist.

If we run any of our commands, such as `dcr-jekyll serve`, and the required Docker image is not in the cache, then Docker will download it.

But if we _never_ run the command, it never gets downloaded.

Which means that we aren't actually 'installing' all the commands we want on our brand new laptop; all we need to do is to ensure that our brand new laptop has Docker on it, as well as all of the `docker-compose.yml` files that drive our applications, and then when we run a command Docker will do what it's good at and download the correct images. All of this means that we get our new laptop up to speed even faster, because we don't install everything in one go.

### A 'mydesktop' Repo

Since everything we need is now in the `dcr-*` style repos, then all we now need is a single package that brings the whole lot together...and that's super simple; just create a package listing whatever `dcr-*` applications you need, and add `bin` definitions for the application name by which you want to run the command.

For example, if we want to have the `dcr-jekyll` and `dcr-mutt` applications available, but we want to run them by simply typing `jekyll` or `mutt`--rather than `dcr-jekyll` and `dcr-mutt`--then we'd have a `package.json` file like this:

```json
{
  "name": "@markbirbeck/mydesktop",
  "version": "0.2.1",
  "description": "My desktop applications, saved as an NPM package.",
  .
  .
  .
  "bin": {
    "jekyll": "./node_modules/.bin/dcr-jekyll",
    "mutt": "./node_modules/.bin/dcr-mutt"
  },
  "dependencies": {
    "dcr-jekyll": "^0.3.0",
    "dcr-mutt": "^0.2.1"
  }
}
```

In short, whatever you put in the `bin` key is the application name you'll use.

Now all that's left to do is push this to GitHub--here's [my desktop](https://github.com/markbirbeck/mydesktop) if you want to take a look--and then you can install your  _entire_ saved desktop with this simple command (or your version of it):

```shell
npm install -g markbirbeck/mydesktop
```

We're making use of the fact that NPM will install from GitHub by default when presented with an `a/b` style module name, which means we won't clutter NPM with our own personal modules.

There's nothing to stop you taking this approach further and creating separate app collections for development tools, communications tools, games, tools you use for particular clients (if you're a freelancer), and so on. It would then just be a case of installing the correct combination for work and home machines.

## Conclusion

It might feel like we've done a lot of work here, but actually we've simply brought together some state-of-the-art technologies to make management of our desktop machines simple, trackable and reproducible; we've used Docker to run the same application on different operating systems, we've used Docker Compose to capture the detail of how each application should run, and we've used Node's package manager to ensure that we can install these valuable snippets into any environment.

In a future post we'll look at how to ensure we can track our data and configuration files across devices.

## Notes

[^1]: Each time I've worked on a Ruby project it has taken an age to get everything set up correctly, whether finding missing gems, getting compilers to work, or a myriad of other issues. I've usually been working with developers who've had their systems set up for ages and can't quite remember how they got there.

[^2]: When I started working on this idea I of course Googled around to see what other people had done in this space. I didn't find anyone trying to turn the whole setup into a load of managed packages, or using Docker Compose to capture the logic, but I did find a few people using Docker to run apps that one would ordinarily install on your laptop. Far and away the best material I read was from Jessie Frazelle; no surprise there...she knows her Docker inside out. In particular her blog post [Docker Containers on the Desktop](https://blog.jessfraz.com/post/docker-containers-on-the-desktop/) is crammed full of really clever ideas, and there are also links to talks that she did on the subject.

[^3]: For years I've been using a Mac for development, and for almost as long, I've been telling myself that one day I'll switch to a Linux desktop like Ubuntu. The logic is simple; I spend so much time dealing with Linux servers within services like AWS that most of the tools and command-line 'phrases' I use are Linux-based. In fact, even though I love using Sublime Text, I use it in Vim-mode so that any new technique I learn can be used anywhere that I'm running Linux, such as when managing servers. So by using Linux everywhere I'm keeping my skills consistent and reusable.