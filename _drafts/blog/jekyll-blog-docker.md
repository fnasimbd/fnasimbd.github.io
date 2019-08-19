---
layout: post
title: "Jekyll Blog with Docker"
modified:
categories: blog
excerpt:
tags: ["ide", "random-notes"]
comments: true
---

This blog is hosted with _GitHub Pages_ (which in turn uses _Jekyll_ server and Jekyll is written in Ruby). Jekyll is a nice tool for its job, however, for years, I had been struggling with a few problems: Ruby environment is so brittle, things break so easily (e.g. a Gem update, different environment, etc.) Moreover, the enviornment is not portable, I want to contirubute (or save drafts) from anywhere: so far, I have been publishing posts from a carefully curated VMWare Ubuntu image (I am a Windows user). Things got even more compounded, since I started using _Docker_: Docker and VMWare can't run simultaneously (as the former uses _Hyper-V_ virtualization!) I found the ideal setup much later and that exploits Docker itself.

_Docker_ comes to mind first whenever you step over on issues with environment; finding the right docker image for Jekyll is what you need to do. So far I am using the `etelej/jekyll` image as the official Jekyll image (`jekyll/jekyll`) has problem exposing the loopback address (localhost or 127.0.0.0).

_Visual Studio Code_ offers extensions that can integrate your workspace with remote environments (e.g. SSH, Docker containers, etc.) All you need to do is install the _Remote Development Extension Pack_ and find the right remote environment for your purpose.

Checkout your GitHub Pages repository to any machine running VS Code and Docker; open the checked out repository in VS Code; create a file called `.devcontainer/devcontainer.json`; set your preferred Docker image name in `devcontainer.json`; press `F1` and choose _Remote-Containers: Open folder in Container_. Now you are effectively writing code targetting the Docker image's environment.


{% if page.comments %}
<div id="disqus_thread"></div>
<script type="text/javascript">
    /* * * CONFIGURATION VARIABLES * * */
    var disqus_shortname = 'fnasim';
    
    /* * * DON'T EDIT BELOW THIS LINE * * */
    (function() {
        var dsq = document.createElement('script'); dsq.type = 'text/javascript'; dsq.async = true;
        dsq.src = '//' + disqus_shortname + '.disqus.com/embed.js';
        (document.getElementsByTagName('head')[0] || document.getElementsByTagName('body')[0]).appendChild(dsq);
    })();
</script>
<noscript>Please enable JavaScript to view the <a href="https://disqus.com/?ref_noscript" rel="nofollow">comments powered by Disqus.</a></noscript>

{% endif %}

