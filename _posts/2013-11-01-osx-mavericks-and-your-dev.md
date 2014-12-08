---
layout:     post
title:      "OS X 10.9 + Your Dev Environment"
date:       2013-11-01
summary:    "Troubleshooting a broken dev environment after upgrading to OSX 10.9"
categories: troubleshooting
---
Every major release of OS X has the potential to really bork your dev environment. The latest release of OS X (10.9 or "Mavericks") caused a few issues for me. Here's what happened and how I fixed it.

After upgrading, I attempted to install a gem from the command line. That didn't go so well:

{% highlight bash %}
ERROR: Failed to build gem native extension.  
/Users/CodyD/.rvm/rubies/ruby-1.9.3-p448/bin/ruby extconf.rb
checking for gmake... no  
checking for make... yes  
 -- /usr/bin/make -f Makefile.embed
checking for main() in -lgit2_embed... *** extconf.rb failed ***  
Could not create Makefile due to some reason, probably lack of  
necessary libraries and/or headers.  Check the mkmf.log file for more details.  You may need configuration options.  
{% endhighlight %}
And then this:

{% highlight bash %}
/Users/CodyD/.rvm/rubies/ruby-1.9.3-p448/lib/ruby/1.9.1/mkmf.rb:381:in `try_do':
 The compiler failed to generate an executable file. (RuntimeError)
You have to install development tools first.  
{% endhighlight %}

When installing a gem, Ruby relies on a slew of libraries provided by XCode, Apple's development environment. And when new versions of OS X are released, XCode is typically updated as well. However, you'll need to manually update XCode's Command Line Tools before you can install new gems:

{% highlight bash %}
xcode-select --install  
{% endhighlight %}

This will prompt you (via the GUI) to download a fresh version of XCode's command line tools. After installation, all should be well again.
