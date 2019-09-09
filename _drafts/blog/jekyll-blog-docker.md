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

This blog is hosted on [_GitHub Pages_](https://pages.github.com) and is built with [_Jekyll_](https://jekyllrb.com) static site generator. GitHub Pages serves only _static pages_; users may either provide their static pages pre-generated or provide templates and configurations for the Jekyll site generator; I opt for the second option. For Jekyll, GitHub Pages conforms to this convention: user creates a public GitHub repository named _github\_userid_.github.io, writes his posts in [Markdown](https://www.markdownguide.org/getting-started), pushes the posts to the repository, that push triggers a Jekyll build and generates the actual static pages to be served. The limiting factor in this case is, however, that user cannot see the posts rendered until it is built by GitHub Pages and made public at the same time. In order to preview posts, user has to create his own [local Jekyll build environment](https://help.github.com/en/articles/setting-up-your-github-pages-site-locally-with-jekyll), preferably equivalent to the GitHub Pages'.

For some time, I curated a VMWare Ubuntu image (I am primarily a Windows user) for Jekyll and Ruby. That setup gave me the necessary isolation from dependency conflicts, and I could preview drafts from that. That setup worked, but I wanted a more portable one so that I could publish posts or preview drafts from anywhere. Even worse, things came to a halt when I started using _Docker_ in my Windows machine: Docker and VMWare can't run together in Windows (as the former uses _Hyper-V_ virtualization, which VMWare is not compatible with!) I reached the ideal setup later, exploiting Docker itself.

**Info.** Jekyll, like GitHub itself, is written in Ruby. For publishing posts, however, no knowledge of Ruby is required. But, as we will see, basic familiarity with Ruby's dependency management (e.g. Gems, Gemfile, Gemfile.lock, Bundler, etc.) helps setting up your local environment and even troubleshooting common issues.
{: .notice--info}

Firstly, I had to find the right [Docker image](https://docs.docker.com/glossary/?term=IMAGE) for Jekyll, with Jekyll and necessary dependencies installed; a container built from that image gives an isolated development environment, without dependency problems. I use the [official Jekyll image](https://hub.docker.com/r/jekyll/jekyll), with tag 3.8.5---that is Jekyll version 3.8.5. The image is built on these conventions: check out your GitHub Pages repository in the host machine, [bind](https://docs.docker.com/storage/volumes/) your checked out repository path to `/srv/jekyll` (so that the Jekyll environment inside container can access it,) [map](https://docs.docker.com/config/containers/container-networking/#published-ports) port `4000` to a host machine port (so that you may access the website generated in the container from the host machine,) and once a container, built from the image, starts properly, your website is built from the `/srv/jekyll` directory by Jekyll and is available at the host's mapped port. 

Next comes building the [Docker container](https://docs.docker.com/glossary/?term=CONTAINER) from the image the right way. Mapping the repository path and the port is straightforward. Additionally, I want Jekyll generate my post drafts (posts not yet ready to be published, stored inside `_drafts`.) But my image starts Jekyll without the `--drafts` switch, omitting draft posts; I needed to augment its [entrypoint](https://docs.docker.com/glossary/?term=ENTRYPOINT) with this command to enable draft generation: `jekyll serve --drafts`. I ended up with the following Docker run command:

```powershell
docker run -v "{path to blog repo}:/srv/jekyll" -p 4000:4000 -it jekyll/jekyll:3.8.5 /bin/bash -c "jekyll serve --drafts"
```

The `jekyll serve` command builds the website from the templates and subsequently starts an HTTP server that runs indefinitely until user interruption. As user interrupts the server (by pressing `Ctrl + C`,) the container terminates with it.

**Warning!** Ideally, `jekyll serve` command starts with _auto-regeneration_ enabled and automatically rebuilds pages as it detects changes in the source directory. But due to some bug, though auto-regeneration is enabled, Jekyll doesn't detect changes in [mapped volumes](https://docs.docker.com/storage/volumes/) in Docker for Windows; hence pages are not rebuilt automatically. Preserving the container (omitting the switch `--rm` in `docker run`, just as I did) and restarting it before previews---restarting containers is fairly fast---serves well as a temporary workaround unless you change any dependencies.
{: .notice--warning}

I want my local build environment to be identical to that of GitHub Pages', ensuring that my locally previewed website looks the same as the one produced at the GitHub Pages server. Mismatch between local and server dependency versions may produce different results. In order to achieve that, GitHub Pages recommends specifying _only_ the `gem "github-pages", group: :jekyll_plugins` gem in your `Gemfile`; all other dependency gems will be resolved transitively, as per the GitHub Pages [versions](https://pages.github.com/versions/).

**Warning!** The gem snapshot file `Gemfile.lock` locks the gem versions at a certain instant in order to avoid future conflicts. The Ruby dependency manager _Bundler_ respects both the Gemfile and the snapshot file while resolveing dependencies; if any discrepency is detected between the two, dependency resolution is aborted. Unversion the snapshot file; otherwise, your local gem versions ships to the repository and gem resolution may fail---and page build too with it---in the GitHub Pages environment as a result even if it succeeds locally. Also, if you feel some dependency version might be causing problem in your local build, delete the snapshot file; everything starts afresh with it.
{: .notice--warning}

**Warning!** In some builds, some hyperlinks may not be accessible from host machine and some resources (e.g. images, css files, etc.) may fail to load too because they are generated with base address `0.0.0.0` instead of `localhost`. That happens mainly due to Jekyll version mismatch; choose the correct Jekyll version if you want to stick to the current configuration or update your configurations according to the Jekyll version if you got the correct Jekyll version.
{: .notice--warning}


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
