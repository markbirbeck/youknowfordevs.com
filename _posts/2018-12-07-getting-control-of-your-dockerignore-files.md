---
layout: post
title:  "Getting Control Of Your .dockerignore Files"
date:   2018-12-07 09:04:01 +0000
comments: true
tags: 
 - docker
---

Every now and then I'll notice a file inside a Docker container that really shouldn't be there. Thankfully it's never been a `.env` file or an SSH key, but even so, any unused file or directory takes up space in the image and makes that image slower to build and pass around. Best practice suggests using the oft-neglected `.dockerignore` file to keep our secrets secret and make sure our Docker images are as lean as possible. But this file then usually becomes a maintenance nightmare as the list of exclusions grows.

This post shows how to control `.dockerignore` by indicating which files we want to *include* rather than those we want to *exclude*.

<!--snip-->

# The Goal

For a while now I've been trying to come up with an easy way to exclude unecessary files from my Docker images. Obviously I want to keep secrets out, and I want to prevent any unused directories from being transferred to the Docker daemon when building--they slow down the build as well as increase the size of the resulting images.

But I also wanted to find a way to exclude files that relate to the build process itself, like `Dockerfile` and `docker-compose.yml`; it's unnecessary clutter if they get transferred to an image and it would be better if they weren't included at all. This is even more important should you want to share an image through public channels like Docker Hub while keeping the source that builds that image internal to your organisation; in this case there may be private information in these build files and you certainly won't want them to be shared.

One last requirement is that when we make a mistake with our ignore rules we want to know about it. The way `.dockerignore` is normally used, if you mistakenly put something like 'git' into your file as a pattern, instead of '.git', you won't know anything about it unless you spot that a lot of data is being transferred to your build.

The rest of this post looks at how we might solve these problems.

# Approach #1: Build From A Single Directory

Most of the approaches I have tried in the last year or so have revolved around different ways of organising the project directories. There are lots of ways a project could be organised, but in general, the approach is to try to keep the source, tests, documentation, generated files, and anything else that is part of the project, separate from each other.

As part of this separation, we also want to ensure that any directories that we don't want to appear inside the Docker image get set as *peers* of the main directory that we want to include. This simply means that rather than having `app > docs` or `app > test` as our directories, we would instead have `app`, `docs` and `test` as top-level directories sitting alongside each other.

By having one directory that contains all of the files that will be needed in a build, and everything else kept safely apart in other directories, we at least start to lower the risk of mistakenly copying something important or unnecessary.

## Setting the Context

Once our directories are organised then it's a simple matter to set the Docker build context to refer to the single directory that should be included in the Docker image. By setting the context to a sub-directory we have ensured that all other directories are excluded, whether they are `.git`, `node_modules`, documentation and tests, and so on.

However, the weakness with this approach is that since the build context must contain *everything* that Docker will need to build the image, that, unfortunately, means we need to place `Dockerfile` in our source directory too.

## Ignoring With `.dockerignore`

It's easy enough to stop the `Dockerfile` from being copied into the image by creating a `.dockerignore` file and then adding 'Dockerfile' to it. The `.dockerignore` file is an 'ignore file' which tells the build process which files to leave out when transferring the context to the Docker daemon. For this situation it could be as simple as this:

```
# In .dockerignore
Dockerfile
```

Don't worry that this could prevent the whole build process from working. In the case of `Dockerfile`, Docker will still transfer it to the daemon to guide the build process, regardless of whether we exclude it or not. But by adding it to our ignore file we stop it going any further through the process; i.e., it won't be available to be copied into the image.

Of course, if we add a `.dockerignore` file to our source directory then that will be available to the Docker daemon, so we now need to ignore that as well:

```
# In .dockerignore
.dockerignore
Dockerfile
```

## Problems With Approach #1

Although this is a nice simple solution, there's still nothing to say that we might not accidentally copy something into the image that we didn't want to. For example, when developing code on our laptop we'll probably have a `.env` file full of database passwords and server names in the source directory. If we forget to exclude the `.env` file as well then it will most likely be copied into our image.

I also prefer not to leave the Docker-related files in the same directory as the source code since it is mixing layers--the source file of our application is now sat alongside the instructions to *build* our application. It's all a bit 'meta'.

# Approach #2: A Richer `.dockerignore` File

So the next step would be to keep the build files out of the main directory, moving them to the root of the project. If we do that then we must make the root of the project the 'build context', and that means we're now exposing all of our files to Docker. To exclude the peer directories we mentioned earlier we would modify the ignore file further, like this:

```
# In .dockerignore
.dockerignore
Dockerfile
dist
docs
test
```

If the code we want to put in our image is in the `app` directory then this `.dockerignore` file will allow that directory to be copied, but will exclude the `dist`, `docs` and `test` directories.

## Problems With Approach #2

But whilst we're excluding tests and documentation, we should also be excluding our `.git` directory (to keep the size down) and other build files, like `docker-compose.yml` and `.gitlab-ci.yml` (to keep things private). Now our ignore file looks like this:

```
# In .dockerignore
.git
.gitlab-ci.yml
.dockerignore
dist
Dockerfile
docker-compose.yml
docs
test
```

This approach of continually growing the ignore file works and you'll find many blog posts that describe exactly what we're doing here. But it soon gets very, very tedious to keep excluding all of the files and directories that are *not* needed in the image, and after a while, you'll find yourself turning a blind eye to any 'harmless' files that are exposed or that get inadvertently copied in.

But that means we're back to square one again because we can *never really be sure* that we haven't copied in something that shouldn't be there, or that the image we've created couldn't be smaller.

Luckily it turns out that there is a much easier way to maintain these exclusions, and it's pretty much foolproof.

# Approach #3: Define *Inclusions*, Not *Exclusions*

As we all know, the `.dockerignore` file contains a list of file patterns to exclude. But it doesn't *just* contain that. The ignore file also contains a list of file patterns to *not* exclude.

Say we have a blog full of Markdown files that we want to exclude from our image because they will be converted to HTML beforehand and are therefore unnecessary at runtime. But let's say also that we still want to include the `README.md` file from our project because it might help anyone looking inside a running container. We could achieve this goal with the following entries in a `.dockerignore` file:

```
# In .dockerignore
# Exclude all Markdown files:
#
*.md

# Now make an exception for one file:
#
!README.md
```

With this file, when Docker starts the build process the only Markdown file that it will pass to the daemon is the `README.md` file.

## Exclude Everything

So why don't we use this feature to our advantage, and begin every `.dockerignore` file with a statement to *exclude everything*? If we start by saying that *nothing* will be sent to the Docker daemon when building an image, it will be almost impossible for us to accidentally include anything that's secret or unecessary.

The pattern that we need in the `.dockerignore` file to ignore everything is simply an asterisk:

```
# In .dockerignore
# Exclude everything:
#
*
```

Now of course, if we exclude everything when we're building the image then it won't just be our secrets that don't get copied in...nothing will! It's like being certain of not losing a race by not entering at all.

So now we need to expand our `.dockerignore` file with information about the files that we *do* want to be included.

And those files should only be those that are mentioned in our `Dockerfile`.

## Only Include Those Files Referenced In `Dockerfile`

Let's assume that we're building a Node app. And let's also assume that we're following best practice in our `Dockerfile` by installing all of the dependencies in one layer and our source code in another layer.

The 'install dependencies' step might look like this:

```
# In Dockerfile
# Copy dependency definitions and then install:
#
COPY package.json .
ENV NODE_ENV=production
RUN npm install
```

while the 'copy source' step could be as simple as this:

```
# In Dockerfile
# Copy our app:
#
COPY app/ .
```

When building the Docker image with this `Dockerfile` the Docker daemon must have access to both the `package.json` file and the `app` sub-directory. And since we've excluded everything at the top of the ignore file, every file or directory referred to in the build process must be explicitly allowed through.

We can easily add these files to our .`dockerignore` file with a 'don't exclude' pattern:

```
# In .dockerignore
# Exclude everything:
#
*

# Now un-exclude package.json and the app folder:
#
!app
!package.json
```

That's it...we're done. The Docker daemon won't receive `.git` or test directories, or `.env`, or `.gitlab-ci.yml`, or `Dockerfile`, or `docker-compose.yml` or anything else.

Now all we have to do is keep the files `.dockerignore` and `Dockerfile` in sync--i.e., ensuring that they both refer to the same paths and files--and nothing else will accidentally get into our Docker images.

One of the many neat features of this approach is that if we forget to add a rule we'll hear about it. If we indicate in our `Dockerfile` that we want to copy in the `README` or the `LICENSE`:

```
# In Dockerfile
# Copy our app and the license:
#
COPY app/ .
COPY LICENSE .
```

we'll get an error at build time unless we add `LICENSE` to the `.dockerignore` file:

```
# In .dockerignore
# Exclude everything:
#
*

# Now un-exclude package.json and the app folder:
#
!app
!LICENSE
!package.json
```

In fact, the patterns that we need in our `.dockerignore` file to specify what we want to *include* are simply the `ADD` and `COPY` entries in our `Dockerfile`, which is much simpler to manage than trying to keep track of what needs to be excluded.

## Problems With Approach #3

Although this is my preferred solution and solves a lot of problems, there are still a couple of weaknesses. Although we'll get notified by the build process if we are missing an important rule from our `.dockerignore` file, we *won't* be notified if our rules are too liberal. So in our example above, if we later decide not to copy the `LICENSE` file when building, there is nothing to tell us that we can remove the pattern from `.dockerignore`.

And also, we still need to remember not to leave anything in the `app` directory that might get copied in. As it happens, the example I gave earlier of a `.env` file (in Approach #1) is now less of an issue since it is best kept in the root directory anyway.

# Conclusion

I've tried a few different solutions to this problem over the years, but I think this 'include file' approach is about as minimal and easy to maintain as it gets. If the notion of 'exclude everything and then un-exclude the files you want' seems a bit quirky just flip things on their head and treat the `.dockerignore` file as if it was an 'include' file--in other words see it as 'only the files listed will be included in the build'.

I think I'd prefer it if this was the default behaviour from Docker, because then we would get both security and minimal image size as the default behaviour, right out of the box. But unless a `.dockerinclude` feature is added to Docker any time soon, then the solution described here is about as good as it will get.