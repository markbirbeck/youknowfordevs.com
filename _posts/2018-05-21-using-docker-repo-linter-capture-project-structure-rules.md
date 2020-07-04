---
layout: post
title:  "Using Docker and Repo Linter to Capture Project Structure Rules"
date:   2018-05-21 16:44:01 +0000
comments: true
tags: 
 - docker
 - repolinter
 - open source
---
One of the things that's frustrating about having lots of open source projects is, well...having lots of open source projects. Some issues are obvious, such as how to keep projects moving along, responding to pull requests, fixing bugs and adding features. But there are plenty of less obvious things to deal with, such as how to keep a whole bunch of projects aligned once you've decided on some best practices.

<!--snip-->

# StandardJS

<a href="https://standardjs.com" style="float: right; padding: 0 0 20px 20px;"><img src="https://cdn.rawgit.com/feross/standard/master/sticker.svg" alt="Standard JavaScript" width="100" align="right"></a>

Let's say we've come across [StandardJS](https://standardjs.com/) (which we have) and we want to apply it to all of our projects (which we do); how do we go about this? Obviously we have to locally clone every project, run StandardJS to reformat the files, run tests on the results, commit the changes, and submit a pull request. And then when we realise that we forgot to add the StandardJS badge to each and every `README` file, we do the whole dance again.

Even if we put to one side (at least for now) the difficulty of making lots of changes to lots of projects, how do we ensure that any new project we create follows whatever new conventions we've adopted?

# Occam's Razor

Take something really simple like an `AUTHORS` file. In a Node project we can either put the [names of contributors into the `package.json` file](https://docs.npmjs.com/files/package.json#people-fields-author-contributors), or NPM will pick up the names of contributors from an `AUTHORS` file if one is present. Which approach is better? Well in my opinion the latter is better because the `AUTHORS` file is a convention that you'll find in many different types of open source projects, not just Node. My personal rule is that if there is a language-*independent* convention that can be followed at no cost then I will always follow it, rather than using a language-specific equivalent. (Think [Occam's Razor](https://simple.wikipedia.org/wiki/Occam%27s_razor).)

<p><a href="https://commons.wikimedia.org/wiki/File:William_of_Ockham.png#/media/File:William_of_Ockham.png"><img src="https://upload.wikimedia.org/wikipedia/commons/7/70/William_of_Ockham.png" alt="Stained-glass window showing William of Ockham"></a><br>By self-created (Moscarlop) - <span class="int-own-work" lang="en">Own work</span>, <a href="https://creativecommons.org/licenses/by-sa/3.0" title="Creative Commons Attribution-Share Alike 3.0">CC BY-SA 3.0</a>, <a href="https://commons.wikimedia.org/w/index.php?curid=5523066">Link</a></p>

But regardless of which way we come down on tracking contributors, the point is that we've thought about a problem, we've made a decision, and the result is a simple rule that we can easily follow in all of our projects (regardless of language)--in my case it's *always have an `AUTHORS` file*. If we can just remember this rule then we can avoid having to think about it each time we start a new project; after all, there's nothing worse than the Groundhog Day moment(s) of going through exactly the same decision-making process all over again, just to arrive at the same conclusion.

All of which means we've moved the problem along a little; whereas before we were having to decide every time we started a new Node project whether to put contributor names in the `package.json` file or an `AUTHORS` file, now the problem is *how do we remember what we decided last time?* At least this problem has the advantage of being the same no matter what issue we are trying to solve...Which license should I use? Which version of Node should I put in my `Dockerfile`? And so on.

I've considered various ways to capture these kinds of rules, once they've been decided on. They could be written in a notebook and kept in the top drawer of your desk. But one criteria I'd like is to make the rules shareable and discussable. Which means that even if the only person discussing the rules was myself *with* myself, it would be handy to have a space where the [advantages and disadvantages of using, say, StandardJS instead of ESLint directly](https://standardjs.com/#i-disagree-with-rule-x-can-you-change-it), could be weighed up. (Not to mention whether or not to use semicolons.) So whilst some kind of shared document would be great, a Git repo would be even better.

# Specifying Rules

Assuming this knowledge is captured in a repository with all the benefits of issue-tracking and comments, what form should the rules actually take?

They could just be written in prose, such as *always use an `AUTHORS` file*, and that would certainly be a great start. But even better would be to find a way to express the rules in such a way that they could be enforced by software. And it's at this point in my research and Googling that I came across [Repo Linter](https://github.com/todogroup/repolinter).

![](https://github.com/todogroup/repolinter/raw/master/docs/images/P_RepoLinter01_logo_only.png)

## Repo Linter

Repo Linter contains a set of generic tests, such as *check that a certain file exists* or *check that a set of files contain the specified text*. These tests can then be used to create more specific *rules* such as *does a README.md file exist?* or *do all of the source files have a copyright message at the top?* Rules are then further combined into *rulesets* to create the exact collection of policies that you want to enforce for your projects.

To illustrate, a ruleset that contains two rules--one to check that there is a license and another to check that there is a `README`--might look like this:

```json
{
  "rules": {
    "all": {
      "license-file-exists:file-existence": [
        "error", {"files": ["LICENSE*", "COPYING*"]}
      ],
      "readme-file-exists:file-existence": [
        "error", {"files": ["README*"]}
      ]
    }
  }
}
```

This is pretty powerful, but Repo Linter goes further and allows us to specify rules that are specific to certain languages (the rules above will apply to all languages). For example, to ensure that all of our Node projects include a `package.json` file we'd add this:

```json
{
  "rules": {
    "all": {
      ...
    },
    "language=javascript": {
      "package-metadata-exists:file-existence": [
        "error", {"files": ["package.json"]}
      ]
    }
  }
}
```

So now we have a way to express our rules in a simple JSON file. All that's left is to apply them against a repo.

# The Answer is Docker (Now...What was the Question?)

The easiest way to get these rules into a reusable form is to create a Docker image that bundles both Repo Linter and a configuration file for some set of specific rules. I've created a basic Docker image that installs Repo Linter and its dependencies, and adds some simple rules to check for a license, `AUTHORS` and `README` files. There are also some instructions to help create new images based on this base, so that more specific rules can be added. The repo is [markbirbeck/docker-repolinter](https://github.com/markbirbeck/docker-repolinter) and the Docker image it generates is in Docker Hub at [markbirbeck/docker-repolinter](https://hub.docker.com/r/markbirbeck/docker-repolinter/).

You can also skip this and go straight to the repo I've set up to do exactly what I've described in this post--capture rules and have a place where they can be discussed. That repo is at [markbirbeck/my-repolinter](https://github.com/markbirbeck/my-repolinter) and it should be very straightforward to copy it and create a Docker image that captures your own policies.

# Where Next?

In a future post I'll look at how we might go about *generating* any files that we detect are missing.