---
layout: entry
title: "Choosing a Blogging Platform"
date: 2012-02-24
comments: true
tags: 
 - archives
 - blogging
 - drupal
 - hyde
 - jekyll
 - octopress
 - static site generation
---

This post originally appeared in 2012 at http://markbirbeck.com/2012/02/24/choosing-a-blogging-platform/. The conclusions and thought processes remain valid, but now I use Netlify (see [Using Disqus with Netlify](/2018/07/20/using-disqus-with-netlify/)).

The great thing about not having blogged for a few years is that I've just had
the opportunity to survey the blogging landscape. If you're drinking *lean start-up* kool-aid or you're a technical blogger, then what I learned may be useful to you.

<!-- more -->

## The Cloud for Everything

The two blogging platforms I used in the past were Blogger (for my posts about
[XForms and internet applications](https://internet-apps.blogspot.com/)) and Drupal (for my company posts). The first thing I knew for certain was that I didn't want to manage my blog infrastructure. After too many holidays interrupted by trying to resolve problems, I spent a number of years putting anything that I could into the cloud.

The initial part of the journey to the cloud involved replacing services that we were already running on our own servers with those provided by others. For example:

* email, calendar, contacts and documents were moved to Google Apps;
* everyday files were placed in Dropbox and Sugar Sync and more system-related material into S3 (usually thanks to the excellent [Cyberduck][]);
* SVN and Trac servers were shutdown in favour of Bitbucket, which later gave way to Google Code and then Github;
* Bugzilla was dropped in favour of LightHouse;
* SugarCRM servers were switched off in favour of Salesforce;
* and mail servers were dropped for Mailchimp.

Then, with lots of servers doing nothing I stopped our contract with WebFusion and shipped the servers back to the office...where of course they then gathered dust because everything was now being moved to RightScale+EC2, Google App Engine and Heroku. Soon the only server left in the office was a flakey DNS box and even that was eventually shut down in favour of DNS Made Easy, and then Route 53. In short, if there was anything that someone else could manage better than us we let them, until our office ended up with a single computer per person and a wi-fi router. Now the only services we managed were web-sites for our customers and ourselves, and these were on servers running at Amazon.

But when I say 'except for our own web-site' that hides a lot. Since we were building web-sites for other companies (albeit with a lot of back-end functionality) it seemed that we really ought to be managing our own. Even though the site was actually just a collection of blog-posts and an enquiry form, it still felt wrong to use something like Blogger or WordPress to convey our ideas and plans to the world, so we continued with Drupal.

## Drupal

In retrospect it's interesting that my blogs hung on for so long without being 'outsourced'. Perhaps it's because Drupal does quite a nice job of serving up good blogs by way of its plethora of modules -- it really is very easy to add tags, comments, avatars, images, OAuth and RSS feeds, and it's also a snap to ensure that your latest pearls of wisdom from Twitter and Facebook appear on your site.

But it's not just how powerful Drupal is; the quality of blogging platforms two years ago really wasn't a patch on what it is today, and that is probably one of the key things that has changed. When you look now at the state-of-the-art with Blogger, Posterous, WordPress and Tumblr, you really have to come up with some pretty solid reasons as to why you would maintain your own database and web front-end.

## Blogger

So I've obviously convinced myself to drop Drupal and the whole self-hosting
approach, but what should I replace it with?

My other blog faired reasonably well on Blogger over the years, so that platform was obviously a candidate. Google have recently revamped the look-and-feel, and it's also easy to import and export information, create new posts from a variety of places, and use external JavaScript components such as [Disqus][]. All of these things make Blogger a strong candidate but in the end the amount of Google branding they add to your blog, coupled with the lack of support for Markdown (more on this below) meant that I decided against it.

However, it's worth stressing that Blogger's ability to easily import and export your posts, as well as its support for Disqus (which means that you can take with you the comments people leave on your blog, should you move platforms), meant I really had to think hard about it. I believe that structuring your use of services in such a way that you can move your data around whenever you want to should be an important part of the lean mindset, and Blogger really helps that.

## Posterous

Right from the start I really liked the feel of Posterous. Highlights for me are that you can create public and private spaces and add new posts from just about anywhere. But before long I hit problems.

The first issue came when I tried to import data from my old blogs. Posterous can't import from Drupal but since it can read from Blogger I tried to use that as an intermediary to get from Drupal to Posterous. Unfortunately it failed, and even after some very prompt and friendly help from Posterous support it still couldn't be made to work.

It shows how much I liked Posterous though that at one point I considered abandoning my old posts in favour of cherry-picking a few...until that is, I
discovered that it was not possible to use Disqus. That was a show-stopper because one of the most important aspects of my new blog is that I should depend as little as possible on any particular platform, and having both posts *and* comments embedded in the platform seemed too risky.

## Tumblr

Tumblr was initially lower down my list than Posterous because of the way that it presents itself as an activity stream and a community, rather than a blog -- I liked it, but it felt like I was going against the flow to use it as a place to write technical articles. But after a little playing around I actually started to like it more than Posterous, and with a few tweaks the longer technical posts could be made to look less out of place.

Tumblr has no problem with Disqus, and as I started to play with it more I discovered more and more references to it in the settings panels of other services that I used, such as Meetup.

But in the end it was ironically Tumblr's biggest advantage for me that led to my *not* using Tumblr.

## Markdown

If your blog mainly includes articles that contain code snippets and embedded screenshots then writing directly in HTML is laborious. Using any of the inline editors that blogging platforms invariably provide doesn't really help either since it involves way too many mouse-clicks (and often the HTML that is generated is a mess). So in this area Tumblr's biggest plus -- at least for me -- is that it supports the use of [Markdown][] to enter articles.

But once I started thinking about my preference for Markdown, and started to think about the kinds of blog-posts I wrote, I realised that being able to control how code snippets and screenshots appear was pretty much the *most important* thing that I needed. And the best tool I'd found for that was [Jekyll][], the static site generator.

## Static Site Generation

Whilst spending time with Tumblr and Posterous I was also toying with the idea of using a static site generator. If you're not familiar with it, the idea is simply to generate a set of static HTML pages from your posts, and then to deploy those pages to a simple web server. Since they are static the pages are delivered very quickly, with little to go wrong. And because they are static, deployment can be as simple as using GitHub Pages or an S3 folder.

Each time I have considered the idea in the past I have rejected it, primarily because the publishing process generally requires you to be able to have access to your development machine. But then you start to wonder whether you would actually ever use any of the additional methods that Blogger, Tumblr and Posterous support for getting your posts online; would I really blog from my phone whilst out with the kids in the park, or during a meal at a restaurant with friends? And if I did, would I send that post to the same blog that I use for discussing test-driven development and big data? It was starting to dawn on me that with all the other benefits that I would get from having a static site, I could live with only being able to deploy from my main work computer.

### Jekyll v. Hyde

Of course given the tortuous tour I'd had through the world of blogging platforms, it was a certaintly that choosing a site generator was not going to be easy either. Jekyll is great, but it's written in Ruby and I now do everything in Python. I've done a little Ruby in the course of constructing Chef scripts but I prefer to focus on as few languages as I can, so the search was on for a Python equivalent to Jekyll, which is how I came across [Hyde][].

### Octopress

However, at the same time that I found Hyde I also came across [Octopress][], a framework that sits around Jekyll sporting a nice theme and many interesting plug-ins. With Octopress it's not only possible to insert code snippets with language-specific formatting, but it's also possible to insert files from your local drive, snippets from gist, excerpts from other posts, and so on. In other words, when it comes to creating a blog about coding, Octporess makes it very easy to manage all the material you need, and to present it nicely.

Hyde is great too, but more effort has been put into making the framework easy to extend, whilst Octopress has seen a lot more effort directed at giving the resulting site a great, clean, look. Of course, if I ever wanted to add my own plug-ins I'd prefer to be with Hyde, but for now I'm going to treat Octopress as a standalone tool that produces nice looking technical sites, and ignore the Python/Ruby issue.

My suspicion is that I haven't finished with blogging platforms just yet, but with my posts stored as Markdown and any comments to the site stored in Disqus -- not to mention other assets such as images and presentations stored in Flickr, Skitch, S3, YouTube, SlideShare, *et. al.* -- I think I'm travelling about as light as I can and should be able to switch pretty easily if something else looks better.

## Conclusion

As services become increasingly specialised it doesn't make sense to manage your own -- far better to spend time in the areas where you can add value. Modern blogging platforms make it very easy to create a site that is as good as anything you could create on your own managed platform.

However, it's still important to think about how easy it would be to shift platforms should the need arise, and for this reason it's important to place resources like comments, images and videos outside the blogging platform.

And if your blog is development-oriented then most of these online services won't really offer the same control over the output of your site as a static site generator like Jekyll coupled with Octopress does.

 [Cyberduck]: http://cyberduck.ch/
 [Disqus]: http://disqus.com/
 [Markdown]: http://daringfireball.net/projects/markdown/
 [Jekyll]: http://jekyllrb.com/
 [Hyde]: http://hyde.github.com/
 [Octopress]: http://octopress.org/