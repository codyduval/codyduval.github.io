---
layout:     post
title:      "To #each His Own"
date:       2012-10-17 12:31:19
summary:    "(or: How I Learned To Stop Worrying and Love Ruby's Enumberables)"
categories: nuby
---
Only after spending 10 weeks hacking away in Java did I really come to appreciate Ruby's built in iterators. If you've spent any time in Java, you'll know that clunky for and while loops are ubiquitous for stepping through collections[1]. For example:

{% highlight ruby %}
for(int i=0; i<list.size(); i++) {  
   Object o = list.get(i);
}
{% endhighlight %}

Ruby, on the other hand, comes loaded with a bunch of very handy iterators, like #map, #select, and #sort in its Enumerable module. But as a Ruby Nuby, it's easy to gloss over these and simply rely on #each for all of your iteration needs. While there's nothing wrong with leaning on #each, spending just a little time learning the methods in Ruby's Enumerable module will lead to more concise and readable code. For example if you wanted to build a string of keys and values from a Hash:

{% highlight ruby %}
def pair_to_string(hash_pairs)  
  hash_pairs.map{|k,v| "#{k} = #{v}"}.join(',')
end  
{% endhighlight %}

####vs

{% highlight ruby %}
def pair_to_string(hash_pairs)  
  output = []
  hash_pairs.each do |k,v|
    output << "#{k} = #{v}"
  end
  output.join ','
end  
{% endhighlight %}
The #map one liner is a big win over the more verbose #each method. If you're just getting started in Ruby, it's worth spending some getting to know its Enumerable module.

[1] - Yes, I know the Java Collections Framework gives you next(), hasNext(), and remove() to let you iterate through a collection.

