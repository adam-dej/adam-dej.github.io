---
layout: post
title: Sigal push-to-deploy using git
categories: film-photography
---

[Sigal](http://sigal.saimon.org/en/latest/) is a simple static gallery
generator. This describes how to use git to deploy it to a remote server.

Prerequisities
--------------

  - You have to want (or already have to have) a sigal gallery.
  - You have to want to use git to deploy it.
  - You have to have a server (or access to one).
  - You have to be a geek (optionally).

Local setup
-----------

Setup your Sigal gallery as usual, just `sigal init` and configure everything,
with 2 important exceptions: don't change photos destination nor source. Check
that your gallery works by building it locally.

Prepare a suitable gitignore (at the very least ignore `_build`) and put your
gallery under git. Please note: It is said that git is not very good when it
comes to dealing with large binary files. In practice, if you are not making
changes to these files and you are just adding new ones (which is the case for a
photo gallery), it works just fine.

Remote setup
------------

This is a bit more interesting part.

First the installation. You will have to install git to your server, and of
course, Sigal. I presume you already have set up a web server or know how to set
up one.

Prepare a bare repository for your gallery.

~~~
mkdir photos.git
cd photos.git
git init --bare
~~~

Git has a way of running custom scripts when important actions occur, such as
pushes to the remote repository. These scripts are called `hooks`. You can read
more about them [here](https://git-scm.com/book/es/v2/Customizing-Git-Git-
Hooks). We will setup a custom `pre-receive` hook. A `pre-receive` is the first
script to run when the remote is receiving a push. It takes a list of references
from stdin and can abort a push by exiting with a non-zero exit status. Here we
will trigger a build of our gallery, with destination to a folder which is
served by the webserver. If anything goes wrong during the build, we will reject
the push.

In the repository folder you have just created you will find several
directories, one of them is `hooks`. Create a new file there, called
`pre-receive` and set its execute bit. Copy this, and modify it to your liking:

~~~
#!/bin/bash

set -e

TMPDIR="<Your prefered tmp dir>"
SRVDIR="<Directory which your webserver serves>"
GITDIR="<Path to the repository directory (/home/example/photos.git)>"

while read oldrev newrev ref
do
    if [[ $ref =~ .*/master$ ]]
    then
        echo 'Deploying gallery...'
        mkdir -p $TMPDIR
        git --work-tree=$TMPDIR --git-dir=$GITDIR checkout -f --quiet $newrev
        cd $TMPDIR
        export LC_ALL=en_US.UTF-8
        export LANG=en_US.UTF-8
        sigal build pictures $SRVDIR
        rm -rf * $TMPDIR
    fi
done
~~~

And that's it. Every time you push to the master branch your gallery will be
built to the `$SRVDIR` folder. If anything goes wrong during the build process,
`set -e` ensures that we will exit with non-zero exit status, and therefore
reject the push.
