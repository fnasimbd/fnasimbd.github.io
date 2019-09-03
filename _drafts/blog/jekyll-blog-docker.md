---
layout: post
title: "Jekyll Blogs with Docker"
modified:
categories: blog
excerpt:
tags: ["jekyll", "ide", "docker", "random-notes"]
comments: true
share: true
---

This blog is hosted with _GitHub Pages_, built on top of [_Jekyll_](http://jekyllrb.com) server and Jekyll is written in Ruby. GitHub Pages is built on the following convention: user creates a public github repository named _userid_.github.io, user writes his posts in Markdown, and GitHub Pages builds the static webpage as user pushes changes to it. The limiting factor here is, however, user cannot see the posts rendered until it is built by GitHub Pages and made public at the same time. User has to create his own local Jekyll build environment to preview posts.

Jekyll is a nice tool for its job, however, the Ruby environment is so brittle: things break easily (e.g. a Gem update, different OS version, etc.) Ideally, locking the Gem versions should do, however, that doesn't always help. An isolated development environment is an absolute necessity. For some time, I curated a VMWare Ubuntu image (I am primarily a Windows user) for Jekyll, and I could publish posts only from that. That setup was not portable at all; I wanted to publish posts or preview drafts from anywhere. Even worse, things came to a halt when I started using _Docker_ in my Windows machine: Docker and VMWare can't run together in Windows (as the former uses _Hyper-V_ virtualization, which VMWare is not compatible with!) I reached the ideal setup later, exploiting Docker itself.

<hr>

Firstly, I had to find the right [Docker image](https://docs.docker.com/glossary/?term=IMAGE) for Jekyll, with Jekyll and necessary dependencies installed. I use the [official Jekyll image](https://hub.docker.com/r/jekyll/jekyll), with tag 3.8.5---that is Jekyll version 3.8.5. The image follows these conventions: [map]() your GitHub Pages repository to the containers `/srv/jekyll` directory, expose port `4000`. A [container](), built from that image, gives an isolated development environment, resolving dependency problems; your page is available on host's mapped port once the container starts properly. 

Next comes, building the right [Docker container](https://docs.docker.com/glossary/?term=CONTAINER) from the image. I override the `jekyll/jekyll` image's [entrypoint](https://docs.docker.com/glossary/?term=ENTRYPOINT) (a command that starts once the container is initialized, and the container terminates when the command finishes); it starts Jekyll without `--drafts` switch; I want the container keep running and Jekyll regenerating the latest changes while I am editing my post drafts. Starting an infinite loop on container startup does the trick.

```
docker run -v "{path to blog repo}:/srv/jekyll" -p 4000:4000 -it jekyll/jekyll:3.8.5 /bin/bash -c "jekyll serve --drafts"
```

**Notice.** Specify only the `gem "github-pages", group: :jekyll_plugins` gem---Ruby term for dependency---in your `Gemfile`; all other gems will be resolved automatically. This way you remain close the the official GitHub pages build environment.
{: .notice--info}

**Warning!** The gem snapshot file `Gemfile.lock` locks the gem versions at a certain moment in order to avoid future conflicts. The Ruby dependency manager _Bundler_ respects both the Gemfile and the snapshot file while resolveing dependencies. Unversion the snapshot file; otherwise, gem resolution may fail in the GitHub Pages environment (even if it succeeds in your local environment) and page build too as a result.
{: .notice--warning}

**Warning!** Some hyperlinks may not be accessible from host machine because it is generated with address `0.0.0.0` instead of `localhost`. That happens mainly due to Jekyll version mismatch; correct your Jekyll version and change configurations accordingly.
{: .notice--warning}

<hr>

My editor _Visual Studio Code_ streamlines the process even further. Its [_Remote - Containers_](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) extension (recently published, still in preview) integrates my workspace with Docker containers: in my setup, I write code targetting the container's enviroment.

**Warning!** Remote Container support in Visual Studio Code is pretty recent. To date, VS Code public edition doesn't support _Alpine Docker_ images. As a result, VS Code remote container is not supported fot the official Jekyll image.
{: .notice--warning}

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

