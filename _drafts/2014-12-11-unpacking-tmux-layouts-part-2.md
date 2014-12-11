---
layout:     post
title:      "Unpacking Tmux Layouts, Part 2"
date:       2014-12-11
summary:    "Building a better API for building custom layouts in Tmux. In Part 2, re-building the layout string in Ruby!"
categories: tmux
---
In [Part 1](), we dove into the Tmux C source code to divine the magical meanings behind each token in the Tmux custom-layout config string.  As a refresher, here's what a config string looks like for a window split into 4 panes:

{% highlight bash %}
➜  ~  tmux list-windows
1: zsh (3 panes) [178x51] [layout d5d2,178x51,0,0[178x25,0,0{89x25,0,0,26,88x25,90,0,27},178x25,0,26,28]] @12 (active)
➜  ~
{% endhighlight %}

Our goal is to build a 


###Custom Layouts
Out of the box, Tmux comes with 5 preset layouts for splitting up your screen: 

* even-horizontal
* even-vertical
* main-horizontal
* main-vertical
* tiled

I've never found much use for these, and luckily, Tmux supports user-defined custom layouts as an alternative.  Instead of using a preset layout, you start with a single pane, and then use a series of command keys to split, adjust, and size each addtional pane. 

Going through this process once or twice is fine, but it quickly gets old if you lose your session and need to start over.  So Tmux provides the ability to output a magical configuration string that you can copy (and later paste) to recreate a window's particular layout. Here's an example of a split window:

![My helpful screenshot](/images/tmux-split.png)

And here are the the values that tmux spits out when you type `tmux list-windows` for the above split:

{% highlight bash %}
➜  ~  tmux list-windows
1: zsh* (3 panes) [178x51] [layout d5d2,178x51,0,0[178x25,0,0{89x25,0,0,26,88x25,90,0,27},178x25,0,26,28]] @12 (active)
➜  ~
{% endhighlight %}

Starting *after* the world 'layout', that's the custom-layout config string.  If you want to start-up Tmux with a similar window split, you'll need to copy that string and paste it into a new session:

{% highlight bash %}
tmux select-layout "d5d2,178x51,0,0[178x25,0,0{89x25,0,0,26,88x25,90,0,27},178x25,0,26,28]"
{% endhighlight %}

Having to cut-and-paste this string seems very... un-Tmuxy.  Wouldn't it be great if there was a more intuitive way to set up panes?

The first step is to figure out how Tmux reads this custom layout configuration string.  Unfortunately, the meaning of these values is completely undocumented.  So we'll have to take a detour into the Tmux source!

###The Checksum
Here's the config string again with the first token highlighted:

<code>
<span style="font-weight: bold; color: red;">d5d2,</span>178x51,0,0[178x25,0,0{89x25,0,0,26,88x25,90,0,27},178x25,0,26,28]
</code>

We can intuit some of these values just by how they are formatted.  For instance, `178x51` is almost certainly the size of my screen (that is, 178 columns wide by 51 rows/line tall). But that first value is pretty mysterious.  It looks to be some sort of hex value, and it turns out its a checksum of the rest of the string.  Here's the relevant code from the Tmux source:

{% highlight c %}
/* Calculate layout checksum. */
u_short
layout_checksum(const char *layout)
{
  u_short csum;

  csum = 0;
  for (; *layout != '\0'; layout++) {
    csum = (csum >> 1) + ((csum & 1) << 15);
    csum += *layout;
  }
  return (csum);
}
{% endhighlight %}

Checksums are typically used to validate the integrity of a packet of data (often when it's being sent over the wire and it might be corrupted along the way).  Here, the hexadecimal value `d5d2` corresponds to everything after the first comma.  It's calculated by iterating through the string character by character (or bit by bit).  As it moves through the string it applies a series of [bitwise operators](http://www.cprogramming.com/tutorial/bitwise_operators.html) and returns a hexadecimal value.  

So if we make any changes to the layout string, we'll need to generate a new checksum that matches the new string. We'll save that for later, but we can cross that token off and move on to the rest.

###The Rest
Here's our config string again with the next group of relevant values highlighted:

<code>
d5d2,<span style="font-weight: bold; color: red;">178x51,0,0</span>[178x25,0,0{89x25,0,0,26,88x25,90,0,27},178x25,0,26,28]
</code>

The next value after the checksum (`178x51`) corresponds to the size of the window that Tmux is running in (columns x rows). And the two numbers after that (`0,0`) are the `x` and `y` offset values.  The offsets tell tmux where on the screen to start drawing the window, starting in the upper left corner.  

<code>
d5d2,178x51,0,0<span style="font-weight: bold; color: red;">[178x25,0,0</span>{89x25,0,0,26,88x25,90,0,27},178x25,0,26,28]
</code>

Moving down the string, we bump into a `[`.  You'll see both curly-braces and square-brackets used in Tmux's layout config string. The square bracket means an "up/down" layout (or draw a horizontal split across the screen).  So after the first bracket, Tmux is going to split the screen into a window 178 columns wide and 25 rows high (with an offset of `0,0`, starting in the upper left).

<code>
d5d2,178x51,0,0[178x25,0,0<span style="font-weight: bold; color: red;">{89x25,0,0,26,88x25,90,0,27}</span>,178x25,0,26,28]
</code>

Next is `{89x25,0,0,26,88x25,90,0,27}` which means "draw two vertical splits inside this first split." Each split is roughly the same size (`89x25` and `88x25`, respectively) and the second split has an offset `90` columns to the right.  The last number in each series (`26` and `27`) is an id used by tmux to keep track the various terminals opened up.  Tmux throws this away when it's parsing the string and setting up a new layout.

<code>
d5d2,178x51,0,0[178x25,0,0{89x25,0,0,26,88x25,90,0,27},<span style="font-weight: bold; color: red;">178x25,0,26,28]</span>
</code>

Finally, outside the closing `}`, we have the other half of the horizontal split.  It's offset 26 rows from the top of the screen.  Again, we can throw away the final value (`28`) as it won't be used when setting up a new session.         

###Gotchas
You'll need to open up the correct number of panes *before* you can send a custom layout string to Tmux. And remember, if you try to edit the string without generating a new checksum, Tmux will reject it.

###Up Next
Now that we know how Tmux parses the layout string, we can start down the road of automating this process.  
