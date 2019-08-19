---
layout: post
title: "Jekyll Blogs with Docker Test"
modified:
categories: blog
excerpt:
tags: ["ide", "random-notes"]
comments: true
---

This blog is hosted with _GitHub Pages_ (built on top of [_Jekyll_](http://jekyllrb.com) server and Jekyll is written in Ruby.) Jekyll is a nice tool for its job, however, for years, I had been struggling with a few problems: Ruby environment is so brittle, things break so easily (e.g. a Gem update, different OS version, etc.) I could publish posts only from my carefully curated VMWare Ubuntu image (I am a Windows user.) That setup is not portable at all; I want to contirubute (or save drafts) from anywhere. Things came to a halt, when I started using _Docker_: Docker and VMWare can't run simultaneously in Windows (as the former uses _Hyper-V_ virtualization, which VMWare is not compatible with!) I reached the ideal setup much later, exploiting Docker itself.

Firstly, I had to find the right Docker image for Jekyll that has Jekyll, with necessary dependencies, installed. So far I am using the [`etelej/jekyll`](https://hub.docker.com/r/etelej/jekyll/) image as the official Jekyll image ([`jekyll/jekyll`](https://hub.docker.com/r/jekyll/jekyll)) has problem exposing the loopback address. Building a container from that image gives an isolated development environment. ~~I usually ignore the Docker image's [entrypoint](https://docs.docker.com/glossary/?term=ENTRYPOINT) (a command that starts once the container is initialized, and the container terminates when the command finishes); I want the container keep running and publishing the latest changes while I am editing my post drafts. Starting an infinit loop on container startup does the trick.~~

```
docker run -v "{path to blog repo}:/app" -w /app -p 4000:4000 etelej/jekyll jekyll serve --host=0.0.0.0 --drafts
```

My editor _Visual Studio Code_ strealines the process even further. Its [_Remote - Containers_](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) extension (recently published, still in preview) integrates my workspace with Docker containers: in my setup, I write code targetting the container's enviroment.

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

