---
layout:     post
title:      "Ruby Symbols Reviewed"
date:       2012-12-06
summary:    "Digging deeper into Ruby Symbols"
categories: nuby
---
I've been spending a few minutes each day diving deeper into Ruby fundamentals. Peter Cooper's [Ruby Reloaded](https://cooperpress.com/rubyreloaded) videos have been an amazing resource in that regard. I'm going to summarize what I'm learning as a way to cement the concepts. First up are **:symbols.**

Symbols are ubiquitous in Ruby, especially if your way into the language was via Rails. You'll often see them as keys in an array of hashes, such as :id in the passed parameters for a User object.

###"Objects with Names"
Ruby demi-god Jim Weirich says that symbols are just "objects with names." Peter Cooper takes that step further by calling symbols "values with names." I think Peter's definition is a little easier to understand. Just as a real-world symbol is a representation for some other real-world meaning or concept (much as an icons exist in GUIs), a :symbol in Ruby is a representation for a value in Ruby. And just as a the meaning of a floppy-disk icon doesn't change as you use Microsoft Word (it always means save), Ruby symbols are meant to be immutable, or unchanging.

###Mutability vs Immutability
Ruby symbols are immutable. Said another way, symbols can't be changed at runtime. Regular Ruby strings can be changed in all sorts of ways (after all, that's what we typically do with text), where a symbol remains forever pointing at the value we set prior to runtime. Strings are for data, symbols are for concepts or identities.

###Efficiency
Symbols are created once and only once at runtime. This means for any given symbol, there will be one location in memory reserved. So if you call :gender a dozen times in your code, it always refers to that single spot in memory. In contrast, a simple string such as "gender" called a dozen times in memory will generate a dozen different pointers and slots. Each new "gender" is an entirely new instance (even though they all look the same).
