---
layout:     post
title:      "Unpacking Tmux Layouts, Part 2"
date:       2014-12-11
summary:    "Building a better API for building custom layouts in Tmux. In Part 2, re-building the layout string in Ruby!"
categories: tmux
---
In [Part 1]({% post_url 2014-12-08-unpacking-tmux-layouts %}), we dove into the Tmux C source code to divine the magical meanings behind each token in the Tmux custom-layout config string.  As a refresher, here's what a config string looks like for a window split into 3 panes:

{% highlight bash %}
➜  ~  tmux list-windows
1: zsh (3 panes) [178x51] [layout d5d2,178x51,0,0[178x25,0,0{89x25,0,0,26,88x25,90,0,27},178x25,0,26,28]] @12 (active)
➜  ~
{% endhighlight %}

Our ultimate goal is to be able to specify a screen layout via a format like YAML.  For example, for the layout above, we might use:

{% highlight yaml %}
layout:
  pane: 50v
  pane: 50v
    pane: 50h
    pane: 50h
{% endhighlight %}

This should tell our program that we want two vertical (`v`) splits each at 50%, with the bottom split again in half horizontally.  To pass this information to Tmux, we'll need to build our own custom-layout string.

###The Layout as a Tree
Here's what we know about the mechanics of a Tmux window split:

* To make a new pane in Tmux, you must "split" the current window into two smaller splits.
* You can split a window either vertically (two boxes on top of one another) or horizontally (two boxes side by side).
* New splits are always nested in within another split.  So any parent window has exactly two child splits. 
* You're not required to split a pane equally, but together, the new panes must fill their parent completely. 

The parent/child relationship of splits mean that a tree would work well as a data structure.  So we'll code up a simple `Node` class to contain each pane split.

 
{% highlight ruby %}
class PaneNode
  attr_accessor :width, :height, :x_offset, :y_offset, :parent, :children, :left_or_right

  def initialize(width, height, x_offset=0, y_offset=0, left_or_right=nil)
    @width = width
    @height = height
    @x_offset = x_offset
    @y_offset = y_offset
    @left_or_right = left_or_right
    @children = []
  end
...

{% endhighlight %}

Then, when we split a window we'll simply add a new child node to the parent split, and adjust its dimensions and offsets accordingly.

{% highlight ruby %}
def split_vertically(percentage)
  first_child_height = ((@height - (@height * percentage)).to_i)
  second_child_height = ((@height - first_child_height).to_i)
  first_child = PaneNode.new(@width, first_child_height, 0, 0, :left)
  second_child = PaneNode.new(@width, second_child_height, 0, 0, :right)

  add([first_child, second_child])

  first_child.x_offset = first_child.parent.x_offset 
  second_child.x_offset = second_child.parent.x_offset 

  first_child.y_offset = first_child.parent.y_offset 
  second_child.y_offset = second_child.parent.y_offset + second_child_height 
end
{% endhighlight %}

Next up, a really long and gnarly method to paramerterize the tree:

{% highlight ruby %}
def tree_as_string(list = [], tree_params = [], brackets = [])
  # Start string with width x height and offset for root node
  if tree_params.empty?
    tree_params << self.params
  end
  unless children.empty?
    children.each do |child|
      list << child
      if tree_params.count != 1 && invisible_pane?
        tree_params << ","
      elsif child.left_or_right == :left
        #if a child and parent have the same width, then it's being
        #split vertically.  Tmux indicates vertical splits
        #with square brackets.
        if (child.width == child.parent.width)
          tree_params << "["
          brackets << "]"
        #otherwise, it's a horizontal split, show that with a curly 
        #brace
        else
          tree_params << "{"
          brackets << "}"
        end
      end
      #if a pane splits in the same direction as its parent, then
      #don't print it (just a weird tmux thing).
      unless child.invisible_pane?
        tree_params << child.params
      end
      #just use 00 for the pane id - tmux throws this away
      #when using a layout string for building a new layout
      if child.window_pane?
        tree_params << ",00"
      end
      #If it's a leaf node, then close the brackets
      if (child.left_or_right == :right) &&
         (child.children.empty?)
        tree_params << brackets.pop
      end
      #keep recursing through the tree, passing any values
      #for the three_params or brackets that its collected
      #along the way
      child.tree_as_string(list, tree_params, brackets)
    end
    #Remember to close any remaining brackets or braces.
    tree_params << brackets.pop
  end
  tree_params.join 
  end
{% endhighlight %}

This long method builds the custom layout string token-by-token by recursing through the tree containing all of the node splits.  

A long method like this violates the good rule of OO design that a method should only do one thing, but in this case it's pretty unavoidable.  There are some tricky edge cases and rules on how Tmux parses the custom layout string.  Much of the code is simply adding brackets or curly braces in the correct places (and making sure they are closed appropriately, as well).

Finally, we need to calculate a checksum of the parameters and prepend it (in hexadecimal) to the front of the final string.

{% highlight ruby %}
class CheckSum
  attr_accessor :config_string

  def initialize(config_string)
    @config_string = config_string
  end

  def csum
    csum = 0
    @config_string.each_char do |character|
      character = character.ord
      csum = (csum >> 1) + ((csum & 1) << 15);
      csum += character 
    end
    csum.to_s(16)
  end
end
{% endhighlight %}

This code mimics the checksum written in the C source. The main `csum` method iterates through the final string character by character, shifts some bits around, then outputs a final checksum value in hex.  We'll append this to the front of the string and BOOM! we have a valid configuration string for any custom layout we desire.  

This project was a fun dive into reading C and outputting the values of a tree into a complex string. I also learned how to include matching brackets into a string by pushing (and popping) them on and off an array.

