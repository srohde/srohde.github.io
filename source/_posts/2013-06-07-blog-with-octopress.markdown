---
layout: post
title: "New Blog with Octopress"
date: 2013-06-07 17:36
comments: true
categories: 
---

I've been [blogging using Workpress](http://soenkerohde.com) since February 2005 and I think Wordpress made the Internet a better place. Luckily I never got hacked but there are just too many stories that sites got hijacked that I finally decided to switch to a static blog. Github pages with [Octopress](http://octopress.org/) seemed like a perfect fit for my needs so here we are.

The setup is dead simple so here my steps:

Created a new Github repo `srohde.github.io` and a bunch of command lines to initialize Octopress:

    git clone git://github.com/imathis/octopress.git octopress
    cd octopress
    gem install bundler
    bundle install
    rake install
    rake setup_github_pages\[git@github.com:srohde/srohde.github.io.git\] 
    rake generate & rake deploy

Bam! The [static site](http://srohde.github.io) is up! Now also commit the sources to the source branch:

    git add .
    git commit -am "initial commit"
    git push origin source

Creating a first post (this very one) couldn't be easier:

    rake new_post\["Blog with Octopress"\]
    subl source/_posts/2013-06-07-blog-with-octopress.markdown
    rake generate & rake deploy

And committing the updated and new sources again:

    git add .
    git commit -am "first post"
    git push origin source

Bamgain (did I just invent that?)! I've got a bunch of topics I wanted to blog about and having a new blogging engine will motivate me to actually do so. That's how devs tick I guess. Cheers!