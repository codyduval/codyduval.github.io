---
layout:     post
title:      "DCI in Rails, Part 2"
date:       2015-01-10
summary:    "The pros & cons of DCI inside Rails"
categories: rails
---
Inheritance.  Encapsulation.  Composition.  Message Passing.  Object Oriented programming and its tenets have become so ubiquitous in modern programming that for many, it is the ONLY way to program.  One of my goals at Hacker School was to try looking at programming problems through a functional lens.  Ultimately, my hope is that some functional concepts will bleed their way into my Ruby programming.

###Learning Clojure
I spent my first week at Hacker School learning the ins-and-outs of Clojure.  Why Clojure?  1) It has a (relatively) small API to learn, and 2) there are a surprising number of "Clojurists" at Hacker School. "Do things at Hacker School that would be a lot harder somewhere else," was one of the first bits of advice given to me on Day 0 of Hacker School.  Having a room full of Clojure experts to pair with flattened the initial learning curve considerably.

###The Challenge: String Compression
Learning to program functionally can be brain bending at first, so I chose small programming challenges to get started.  Here's my second program, which takes a string input from a user and compresses it by removing duplicate characters (eg., turn "aaaaaabbbcccdd" into "6a4b3c2d").     

{% highlight clojure %}
(ns string-compressor.core
  (:gen-class))

(defn get-string
  "Gets a user inputted string to be compressed, for example
  'aaaabbbcccdddeeef'. Returns the user inputted string."
  [] 
  (println "Enter string: ")
  (let [string-input (read-line)] string-input))

(defn split-by-like-chars
  "Take a user inputted string and split into a list of lists of 
  similar characters. Eg: ((aaa) (bbb) (ccc) (dd) (ee) (f))." 
  [user-string]
  (partition-by identity user-string))

(defn count-chars-by-group
  "Take a list of character lists and count the number of characters
  in each list.  Returns a list of maps of each character and its count,
  eg ({a 3} {b 3} ...)."
  [list-of-character-groups]
  (flatten (map (fn [group] {(first group) (count group)})
                list-of-character-groups)))

(defn hashmaps-of-counts-to-single-string
  "Take a list of character counts and convert into a single string. 
  Returns a string like 'a3b3c2d1'"
  [list-of-hashmaps]
  (apply str (flatten (map concat list-of-hashmaps))))

(defn compare-compressed-to-original
  "Compare the length of a compressed string to the original string and
  return which ever is shorter."
  [original compressed]
  (if (< (count original) (count compressed)) original compressed))

(defn compress-string
  "Compresses a user entered string like 'aaaabbccddeeeeff' and returns 
  a string that replaces repeat characters with counts, eg. 'a4b2c2d2e4f2'"
  [user-string]
  (-> user-string split-by-like-chars count-chars-by-group hashmaps-of-counts-to-single-string))
  
(defn go
  []
  (let [user-string (get-string)]
    (let [compressed-string (compress-string user-string)]
    (println "You said:" user-string)
    (println "Compressed this is:" compressed-string)
    (println "The optimal compressed string is:" (compare-compressed-to-original user-string compressed-string)))))

(defn -main
  []
  (go))
{% endhighlight %}

###Breaking It Down
Compressing a string is a relatively simple task, but, like most programs, it still comes down to a series of steps.  To get started, I sketched these steps out:

1. Get a string from the user.
2. Split it up by similar characters.
3. Count the characters in each group.
4. Create a new string with the counts.
5. Compare this compressed string to the original to see if it's shorter.
6. Print the compressed string to the screen.

###Start at the Bottom
I've noticed that for most Clojure programs, the bottom of the program is a good place to start when trying to figure out what it does. In my program, I wrote a bunch of tiny functions to mirror the steps above.  Then, in the function below I string them together using Clojure's "thread-first" macro (the `->`).

{% highlight clojure %}
(defn compress-string
  [user-string]
  (-> user-string split-by-like-chars count-chars-by-group hashmaps-of-counts-to-single-string))
{% endhighlight %}

The thread-first macro is a nicety to make this function more readable. Without the `->` macro, I would nest each function like so:

{% highlight clojure %}
(defn compress-string
  [user-string]
  (hashmaps-of-counts-to-single-string(count-chars-by-group(split-by-like-chars(user-string))))
{% endhighlight %}

Instead of reading the program from the inside-out like the Clojure compiler, the thread-first macro lets you read from a more natural left-to-right.

###Forgetting State
Right off the bat, I really liked letting my brain "forget" about state when composing the program.  In object oriented programming, objects encapsulate both state and methods.  In contrast, functional programming is stateless (or should be, at least), so you're not spending brain cycles keeping track of variable assignment as you compose your program.  

If I were programming a string compressor in Ruby, I'd probably store the initial string in an instance variable, mutate it via some sort of `Compressor` class, and then return the changed variable. Instead, my Clojure program has a function which simply returns a new (compressed) version of the string.  No variable was ever assigned and neither I or the compiler needed to keep track of state.

###No Side Effects
Mary Rose Cook has [a great blog post](http://maryrosecook.com/blog/post/a-practical-introduction-to-functional-programming) on what really defines functional programming: the *absence of side effects.* Data is changed in a function and then returned; there is no state shared between objects.  There are many benefits to not sharing state, the biggest is perhaps the elimination of bugs that arise due to flow control. A stateless program is also free to run concurrently - that's a big win in today's multi-core world. 
