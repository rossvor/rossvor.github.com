---
layout: post
title: "A hacky markdown live preview"
description: "How to use markdown live preview with any editor"
category: scripting
tags: [kate,markdown,inotify,uzbl]
---
{% include JB/setup %}

I use [kate](http://kate-editor.org/) for my day-to-day scripting and note taking. It has markdown highlighting but as of right now there's no way to have a live preview with it. Find below a hackish solution to this problem using [uzbl](http://www.uzbl.org/) browser, inotify watcher and a fifo pipe.

## 1. Things you'll need

* uzbl-browser
* inotify-tools

In arch you can do:

~~~
# pacman -S uzbl-browser inotify-tools
~~~

You'll also need some sort of markdown renderer. [Kramdown](https://github.com/gettalong/kramdown) is one of them. To get it, if you have [rubygems](https://wiki.archlinux.org/index.php/ruby#RubyGems), do :

~~~
$ gem install kramdown
~~~

## 2. Scripts


### watcher.sh
Create a new script, call it watcher.sh in your ~/bin or your favourite place to store user scripts.

~~~sh
#!/bin/sh
pipe=/tmp/testpipe

style='bootstrap.min.css'	# set path to your css stylesheet here if necessary

styletag="<meta charset=utf-8><link href=$style rel=stylesheet>"

trap "rm -f $pipe $1.html" EXIT		# cleanup

if [[ ! -p $pipe ]]; then
    mkfifo $pipe
fi

if [[ ! -e $1.htm ]]; then
    echo $styletag > $1.html
    kramdown $1 >> $1.html
fi

# watch in subshell for modifications
(while true; do
  inotifywait -e modify $1	# set up inotify to monitor our file for modifications
  echo $styletag > $1.html
  kramdown $1 >> $1.html	# use kramdown to render markdown to html
  echo reload > $pipe		# send the command to uzbl through the pipe
done &)

# keep the pipe open
cat > $pipe
~~~

If we don't include `` <meta charset=utf-8> ``, browser will default to a wrong charset and some characters will display incorrectly.

We check if pipe was created and if not we create it with `` mkfifo ``.
Same with temporary html file. We add our tags to the beginning and add the rest with kramdown.

Next we create a subshell with ``inotifywait -e modify $1`` which blocks execution of the rest of the loop untill file is modified. Recreate our temporary html file with ``echo $styletag > $1.html`` and ``kramdown $1 >> $1.html`` and we send our command "reload" to our pipe which will be read by uzbl-browser.

Next bit is a bit of tricky one, basically we have to keep pipe open for writing, otherwise after first ``echo reload`` pipe will close, so we use a dummy cat for that ``cat > $pipe``.

### launchuzbl.sh
Now create launchuzbl.sh. 

~~~sh
#!/bin/sh
pipe=/tmp/testpipe

sleep 1
uzbl-browser -c - <$pipe $1
~~~

We have to launch uzbl-browser with a bit of a delay before we set up our pipe, otherwise it fails to respond to piped commands. I'm still unsure of the reason for this idiosyncrasy.


### previewmd.sh
~~~sh
#!/bin/sh
launchuzbl.sh $1.html & watcher.sh $1
~~~
Just a wrapper script and the actual script user is going to call.


## 3. Result
To call use:

~~~
$ previewmd.sh mymarkdownfile.md
~~~
It will launch non-interactive uzbl-browser window.
Then open your file in editor and everytime you save it will automatically reload and update the browser window.

Here's how it should roughly look with kate on the left and uzbl-browser on the right:
![screenie]({{ site.url }}/assets/img/screenie.jpg)




