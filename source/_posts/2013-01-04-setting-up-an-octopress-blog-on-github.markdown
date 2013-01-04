---
layout: post
title: "Setting up an Octopress blog on github from OSX"
date: 2013-01-04 22:11
comments: true
categories: [Octopress, github, ruby, OSX, homebrew]
---
I just installed [Octopress](http://octopress.org) to deploy a blog to github on my OSX system and found this process not exactly smooth - so I decided to write it down.
 
## Installing ruby using homebrew

Being more of a Java guy who has never actually worked with ruby, installing it on my OSX system (Mountain Lion to be precise) was a bit tricky. I am going to break down the necessary steps here, hoping I forgot nothing. By the way: I'm using [homebrew](http://mxcl.github.com/homebrew/).

But, OSX ships with ruby, you say. That is correct, but this is a dated version (1.8.7 on my system) which is not compatible with Octopress: They require 1.9.3. Thus, we need a current ruby installation to replace the included one. So:
    $ brew install ruby
But now, doing `ruby --version` still outputs `ruby 1.8.7 (2012-02-08 patchlevel 358) [universal-darwin12.0]`. What went wrong? If you look at `/etc/paths` you will notice, that `/usr/local/bin` (the directory homebrew symlinks its binaries into) is at the bottom, making it lose against system defaults (the preinstalled ruby is located in `/usr/bin/` which is higher up in the list). I ended up editing `/etc/paths` to include `/usr/local/bin` at the top, as pointed out [here](http://stackoverflow.com/a/8731098/537738). Keep in mind that this will generally change precedence of homebrew binaries vs the preinstalled ones - but you might want this, anyways.

Now, we get 
```
$ ruby --version
ruby 1.9.3p362 (2012-12-25 revision 38607) [x86_64-darwin12.2.0]
```
Nice!

An alternative would have been to use RVM or rbenv to install ruby. But RVM is not available in homebrew and rbenv requires "the real gcc" which does not ship with XCode anymore (they use LLVM/clang now). So I decided to use homebrew all the way.

### But wait, there's more!
Ruby works fine by now, but there is still something left to do if we want to install gems (which is what we will need to do in order to get Octopress up and running). Using the current setup, if you install a gem like this:
    $ gem install bundler
And try to execute it using
    $ bundler
you will note that the binary is not found. Again, the `PATH` is the culprit: the gem-directory is not on the path. Homebrew helpfully says:
```
$ brew info ruby
[...]
==> Caveats
NOTE: By default, gem installed binaries will be placed into:
  /usr/local/Cellar/ruby/1.9.3-p362/bin

You may want to add this to your PATH.
```
But I feel that adding the directory containing the ruby version to the path is not a particularly good idea. Instead, I followed the advice [here](http://superuser.com/a/527534/83513) and added the following line to my `~/.bash_profile`
    export PATH=$(cd $(which gem)/..; pwd):$PATH
In a new Terminal session, you should now get something like
```
$ echo $PATH
/usr/local/Cellar/ruby/1.9.3-p362/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin:/opt/X11/bin
```

To install ruby on linux, you will probably just do something like
    sudo apt-get install ruby

## Installing Octopress
For installing Octopress, I followed the [directions](http://octopress.org/docs/setup/) on their site starting from "Setup Octopress".

After that, I did what they said [here](http://octopress.org/docs/deploying/github/) in order to deploy to github pages. What I found irritating: You actually have to create a repo on github called something like "myfantasticblog.github.com" if you want your fantastic blog to be available via `http://myfantasticblog.github.com` lateron. Apparently, github uses some magic with the repo name here.

After making the first deployment, I actually saw - nothing. This was because I did not have the required gems on my path (see above) and `rake generate` dit not actually generate any files. And I seem to have missed the error messages somehow.

Everything else went pretty smoothly.

## Useful Octopress commands
There are some commands that you will need to use regularly:

* `rake generate` generate the html pages that can then be deployed
* `rake deploy` deploy the generated page to github
* `rake preview` run a local webserver you can use to preview your page (access via [http://localhost:4000]). This is really nice, as it automatically updates. So, if you just leave it running and update a post, you can just refresh your browser immediately.
* `rake new_post` create a new posting (a markdown file per default). The file is saved into the `source/_posts/` subdirectory
