---
layout:     post
title:      "CSS Performance & Organization"
date:       2013-01-08
summary:    "Getting smarter about the DOM and CSS."
categories: nuby
---
Shay Howe is publishing a series of (free) blog posts on advanced HTML & CSS best practices. I just read the first entry and there were tons of useful nuggets.

###Object Oriented CSS
In order to increase the re-usability of CSS code, Shay advocates an Object Oriented CSS methodology for organizing your style-sheets. The two key principles include 1) separating structure from skin and 2) separating content from container.

By separating structure from skin, you abstract the layout of an element away from the theme of a site. This allows other styles to easily inherit without conflict.

And by separating content from the container, one removes the dependency of a parent element nesting children elements. In other words, a heading should look like a heading regardless of the container its in. This means diligent use of default styles (and then extending those as needed).

{% highlight html %}
<div class="alert alert-error">  
   <p class="msg">...</p>  
</div>  
{% endhighlight %}

###Modular and Scalable CSS
Shay borrows from Jonathan Snook's Scalable and Modular Architecture for CSS (aka the SMACCSS book). He promotes breaking styles up into five categories: 

* Base
* Layout
* Module
* State
* Theme

This system of organization is an alternative to the OO method (above), but achieves the same goal of re-usability and DRY, unduplicated code.

###Other Best Practices
Instead of long chains of CSS selectors (eg `header nav ul li a`), use short class names instead (eg `.primary-link`)

Don't prefix class selectors with an element (eg `article.feat-post` vs just `.feat-post`)

Stay away from using ID selectors where possible. They are overly specific and can't be used more than once.

Look for repetitive code in your CSS, and combine class names with a comma where appropriate.

Reduce HTTP requests by combining your CSS sheets into one file.
