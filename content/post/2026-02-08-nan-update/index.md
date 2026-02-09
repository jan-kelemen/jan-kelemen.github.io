---
author: Jan Kelemen
date: '2026-02-08T00:00:00Z'
tags:
- hugo
- crystalspace
- reposurgeon
title: NaN update
---
Breaker one-nine, this here's the Rubber Dax. You got a copy on me Love Machine?

{{< youtube "Sd5ZLJWQmss" >}}

While I usually write about my game engine [jan-kelemen/niku](https://github.com/jan-kelemen/niku), this post will be a little different.
This is Not A Niku update. If you came for IEEE 754 NaN, sorry, there will be no IEEE 754 either.

# Software archeology
> If someone knows how to convert a huge SVN repository to Git in a history preserving way, please let me know.

[Crystal Space 3D SDK](https://web.archive.org/web/20260203063216/http://www.crystalspace3d.org/) is now available on Git.
The source is now archived under the GitHub organization [crystal-space-archive](https://github.com/crystal-space-archive).

I expect that this will be a one-time effort, due to a now 7 year old, quote from one of the main project maintainers:
> effectively dead and has been for a good number of years

As far as I know, this is the most complete Git conversion. There was [crystalspace/CS](https://github.com/crystalspace/CS), but that's missing the last couple of years of development,
and doesn't include other accompanying projects. [sarahcrowle/crystalspace](https://github.com/sarahcrowle/crystalspace) was just a trunk snapshot.

If you want to know more about Crystal Space, here is a talk from the original author:
{{< youtube "mVArZ6l_xqQ" >}} 

## But why?
I had some ideas with this project, but its source code was hosted on SourceForge.
Working with SVN is one of the skills I would like to avoid in life.
Cloning it with `git svn` is hopeless, it takes more than a day and does nothing in the end.

I've ended up using the [reposurgeon](https://www.catb.org/~esr/reposurgeon/) tool for the repository conversion.
Overall, the code was split on two locations [1](https://sourceforge.net/p/crystal/code/HEAD/tree/), [2](https://sourceforge.net/p/cel/code/HEAD/tree/) with multiple SVN projects inside.
Due to the different principles of how SVN and Git version controls work, I've had to split each SVN subproject into its own Git repository.

The nice thing about `reposurgeon` is that the conversion steps are repeatable and can be incrementally improved.
It downloads the whole upstream repository dump and works on the dump files.
Though the dump file is 40GB in size, so going through that takes 20 minutes, it could be worse.

The general approach for this was to extract the streams of individual projects from the main dump and incrementally improve on the converted history.
Since the project in the SVN was fairly regular in structure, `reposurgeon` did most of the work automatically.
Most of the manual work was related to fixing up generated tags or removing things that didn't make sense to preserve.

One of the more tedious parts was to create an author mapping from SVN usernames to Git style with a name and email.
During ~17 years of development, the project accumulated quite a few developers. 
That didn't seem like a fun activity to figure out by hand.

But this is an open source project, surely the statistical token soup can do that.
Nope, I've tried Gemini. It generated a mapping that mapped only a couple of authors correctly. 
On 98% of emails appended a `@users.sourceforge.net` to the original username, fair, but lazy.
The names were straight up incorrect.

So I opened `gitk --all` in [crystalspace/CS](https://github.com/crystalspace/CS) and in my converted repository and started scrolling.
That mapping should be correct since it was done by the original maintainers.
I've tried to figure out the missing ones from the SourceForge profiles.

Before even attempting this conversion project, I've tried to compile the original SVN sources.
Here is a screenshot of the example application built with Crystal Space.
![walktest.exe](cs.png#center)

I've tried most of the demo applications that are compiled with the project.
They do work, but I've had some crashes. 
I'll assume those were caused by me running it in a VM.

If you want to do it on your own, you'll have to download an installer for third party dependencies from [WebArchive](https://web.archive.org/web/20160329074613/http://www.crystalspace3d.org/main/Download).
A period correct compiler is also required, I've used Visual Studio 2010 in a Windows 7 virtual machine.
If that sounds too sketchy, you could also try building the installer on your own from [source](https://github.com/crystal-space-archive/CSlibs).
Let me know if you figure out how.

PS: The installer is fine. It just has 10 year old CVEs.

# Now self-hosted too!
This blog is now hosted also on my new domain: [jk.rubberdax.xyz](https://jk.rubberdax.xyz).

I've also migrated from the ancient version of Jekyll, which the GitHub Pages infrastructure requires to the [Hugo](https://gohugo.io) static site generator.
Converting from Jekyll to Hugo was quite an easy process. 
Most of the Markdown conversion was done by the `hugo import jekyll` command.

Centering the images in the posts was the hardest part of that.
I guess it's obvious that I'm not doing a lot of web development.
```css
img[src$='#center']{
  display: block;
  margin: 0.7rem auto;
}
```

Jekyll was annoying me for quite some time already. 
I don't use Ruby and installing the whole ecosystem just for a static site generator, and getting things to run locally is something I've always had problems with.
To be fair, I don't use Go either, but at least it's an up to date version and I can update it as often as I want.

This was also an opportunity to fix a lot of the things that were previously broken, featuring a favicon and syntax highlighting.
The site should be more responsive on mobile devices, too.
During the migration, I took care that the links to the previous posts still work, so you can revisit them.

Currently, the [jan-kelemen.github.io](https://jan-kelemen.github.io) and this site are identical in content.
I've added GitHub Actions workflows that publish the page to the GitHub infrastructure and package the site for the self hosted version.

## Hosting
The blog is hosted on the cheapest VPS that [Hetzner](https://www.hetzner.com) currently offers.
This should be adequate for serving a static site with low traffic.

As an honorable mention, I did use this blog post as a starting point on setting up the server: [Moving from GitHub pages to self-hosted](https://belief-driven-design.com/moving-from-github-pages-to-self-hosted-ab1231fb7fa/).
From there, I've created some self inflicted learning opportunities.

The usual approach of rotating NGINX logs with a `logrotate` rule goes something like this:
```
/var/log/nginx/*.log {  
    monthly  
    missingok  
    rotate 12  
    compress  
    delaycompress  
    notifempty  
    sharedscripts  
    postrotate  
        docker inspect -f '{{ .State.Pid }}' nginx-server | xargs kill -USR1  
    endscript
}
```

The `postrotate` script should notify the NGINX server to reopen file handles to the log files.
Well, I've decided not to run NGINX in Docker. 
It's in a rootless container running under a user that has no sudo rights.

This also implies that the `logrotate` job that runs as `root` doesn't see the container, but it should notify NGINX after rotating the log files.

The genius solution here was to run the `logrotate` job in a systemd user unit on a timer.
```
[Service]
Type=oneshot
ExecStart=logrotate --state %h/logrotate.status %h/logrotate.config
ExecStopPost=podman kill --signal USR1 nginx-server
```
Since the systemd user unit runs as the same user that owns the container, it can also notify the container to rotate the log files.

# Final words
Thanks to the beta testers for finding out that my DNS entries are broken.

I should be returning in the next post to the usual game engine updates.
I've started working on the editor application, but it's currently in a state of opening a blank application window. 
I'm taking the time to review stuff written before and refactoring as needed, so things are going a bit slower.

Yeah, we definitely got the front door, Good Buddy. Mercy sakes alive, looks like we've got us a convoy.
