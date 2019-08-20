---
layout: post
title: "Jekyll Blogs with Docker"
modified:
categories: blog
excerpt:
tags: ["jekyll", "ide", "docker", "random-notes"]
comments: true
---

This blog is hosted with _GitHub Pages_ (built on top of [_Jekyll_](http://jekyllrb.com) server and Jekyll is written in Ruby.) Jekyll is a nice tool for its job, however, the Ruby environment is so brittle: things break easily (e.g. a Gem update, different OS version, etc.) Ideally locking the Gem versions should do, however, that doesn't always help. An isolated development environment is an absolute necessity. For some time, I curated a VMWare Ubuntu image (I am primarily a Windows user) for Jekyll, and I could publish posts only from that. That setup was not portable at all; I wanted to publish posts or preview drafts from anywhere. Even worse, things came to a halt when I started using _Docker_ in my Windows machine: Docker and VMWare can't run together in Windows (as the former uses _Hyper-V_ virtualization, which VMWare is not compatible with!) I reached the ideal setup later, exploiting Docker itself.

Firstly, I had to find the right [Docker image](https://docs.docker.com/glossary/?term=IMAGE) for Jekyll that has Jekyll installed, with necessary dependencies. So far, I am using the [`etelej/jekyll`](https://hub.docker.com/r/etelej/jekyll/) image (I also tried the [official Jekyll image](https://hub.docker.com/r/jekyll/jekyll), however, it has a problem exposing the loopback address: all hyperlinks in the generated pages point to the container address, instead of loopback. The problem is most likely due to the Jekyll version: `etelej` uses `3.0.5` and official uses `3.8`. The same issue reproduces even in the `etelej` image if its Jekyll version is update to `3.8`. Passing the intended Jekyll version as environment variable might resolve it.) It follows these conventions: [map]() your GitHub Pages repository to the containers `/app` directory, set `/app` as current workspace, expose port `4000`, and you can access your page on hosts mapped port after container starts. A [container](), built from that image, gives an isolated development environment, resolving dependency problems.

Next comes, building the right [Docker container](https://docs.docker.com/glossary/?term=CONTAINER) from the image. I override the `etelej/jekyll` image's [entrypoint](https://docs.docker.com/glossary/?term=ENTRYPOINT) (a command that starts once the container is initialized, and the container terminates when the command finishes); it starts Jekyll without `--drafts` switch; I want the container keep running and Jekyll regenerating the latest changes while I am editing my post drafts. Starting an infinite loop on container startup does the trick.

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

