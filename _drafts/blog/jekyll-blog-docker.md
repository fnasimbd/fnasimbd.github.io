---
layout: post
title: "Jekyll Blogs with Docker"
modified:
categories: blog
excerpt: "Building Jekyll blogs with Docker."
tags: ["jekyll", "ide", "docker", "random-notes"]
comments: true
share: true
---

This blog is hosted on [_GitHub Pages_](https://pages.github.com) and is built with [_Jekyll_](https://jekyllrb.com) static site generator. GitHub Pages serves only _static pages_; users may either provide their static pages pre-generated or provide templates and configurations for the Jekyll site generator; I opt for the second option. For Jekyll, GitHub Pages conforms to this convention: user creates a public GitHub repository named _github\_userid_.github.io, writes his posts in [Markdown](https://www.markdownguide.org/getting-started), pushes the posts to the repository, that push triggers a Jekyll build and generates the actual static pages to be served. The limiting factor here is, however, that user cannot see the posts rendered until it is built by GitHub Pages and made public at the same time. In order to preview posts, user has to create his own [local Jekyll build environment](https://help.github.com/en/articles/setting-up-your-github-pages-site-locally-with-jekyll), preferably equivalent to the GitHub Pages'.

For some time, in order to avoid dependency issues, I curated a VMWare Ubuntu image (I am primarily a Windows user) for Jekyll and Ruby as an isolated development environment, and I could preview drafts only from that image. That worked, but I wanted a more portable setup so that I can publish posts or preview drafts from anywhere. Even worse, things came to a halt when I started using _Docker_ in my Windows machine: Docker and VMWare can't run together in Windows (as the former uses _Hyper-V_ virtualization, which VMWare is not compatible with!) I reached the ideal setup later, exploiting Docker itself.

**Info.** Jekyll, like GitHub itself, is written in Ruby. For publishing posts, however, no knowledge of Ruby is required. But, as we will see, basic familiarity with Ruby's dependency management (e.g. Gems, Gemfile, Gemfile.lock, Bundler, etc.) helps setting up your local environment and even troubleshooting common issues.
{: .notice--info}

<hr>

Firstly, I had to find the right [Docker image](https://docs.docker.com/glossary/?term=IMAGE) for Jekyll, with Jekyll and necessary dependencies installed. I use the [official Jekyll image](https://hub.docker.com/r/jekyll/jekyll), with tag 3.8.5---that is Jekyll version 3.8.5. The image follows these conventions: [map]() your GitHub Pages repository to the containers `/srv/jekyll` directory and map port `4000` to a host machine port, and your page is available on host's mapped port once the container starts properly. A container built from that image gives an isolated development environment, without dependency problems.

Next comes building the right [Docker container](https://docs.docker.com/glossary/?term=CONTAINER) from the image. I want Jekyll regenerating the latest changes while I am editing my post drafts, so I override the `jekyll/jekyll` image's [entrypoint](https://docs.docker.com/glossary/?term=ENTRYPOINT) because it starts Jekyll without the `--drafts` switch, omitting draft posts. Docker entrypoints are commands that starts once the container is initialized, and when the command finishes the container terminates with it. I also want the container keep running during my edits; `jekyll serve` command starts an HTTP server that runs indefinitely until user interruption.

```powershell
docker run -v "{path to blog repo}:/srv/jekyll" -p 4000:4000 -it jekyll/jekyll:3.8.5 /bin/bash -c "jekyll serve --drafts"
```

**Notice**. Ideally, when `--drafts` switch is enabled, Jekyll automatically rebuilds pages as it detects changes in the source directory. In Docker Windows [mapped volumes](https://docs.docker.com/storage/volumes/), however, Jekyll doesn't detect changes due to some bug; hence pages are not rebuilt automatically. Preserving the container (omitting the switch `-rm` in `docker run`) and restarting it after edits---restarting containers is fairly fast---serves well as a temporary workaround unless you change any dependencies.
{: .notice--info}

We want our local build environment identical to GitHub Pages', ensuring that our locally previewed website looks the same as the final one. In order to achieve that, GitHub Pages recommends specifying only the `gem "github-pages", group: :jekyll_plugins` gem in your `Gemfile`; all other transitive dependency gems will be resolved automatically.

**Warning!** The gem snapshot file `Gemfile.lock` locks the gem versions at a certain instant in order to avoid future conflicts. The Ruby dependency manager _Bundler_ respects both the Gemfile and the snapshot file while resolveing dependencies; if any discrepency is detected between the two, dependency resolution is aborted. Unversion the snapshot file; otherwise, your local gem versions ships to the repository and gem resolution may fail---and page build too with it---in the GitHub Pages environment as a result even if it succeeds locally. Also, if you feel some dependency version might be causing problem in your local build, delete the snapshot file; everything starts afresh with it.
{: .notice--warning}

**Warning!** In some builds, some hyperlinks may not be accessible from host machine and some resources (e.g. images, css files, etc.) may fail to load too because they are generated with base address `0.0.0.0` instead of `localhost`. That happens mainly due to Jekyll version mismatch; correct your Jekyll version and update configurations accordingly.
{: .notice--warning}

<hr>

My editor _Visual Studio Code_ streamlines the process even further. Its [_Remote - Containers_](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) extension (recently published, still in preview) integrates my workspace with Docker containers: in my setup, I write code targetting the container's enviroment.

**Warning!** Remote Container support in Visual Studio Code is pretty recent. To date, VS Code public edition doesn't support _Alpine Docker_ images. As a result, VS Code remote container is not supported for the official Jekyll image.
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

