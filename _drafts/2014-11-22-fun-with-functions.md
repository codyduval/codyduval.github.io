---
layout:     post
title:      "Fun with Functions and Clojure"
date:       2014-11-22
summary:    "Cargo culting Clojure and the joy of functional programming."
categories: clojure
---
Inheritance.  Encapsulation.  Composition.  Message Passing.  Object Oriented programming and its tenents have become so ubiquitous in modern programming that for many, it is the ONLY way to program.  One of my goals at Hacker School was to try looking at programming problems through a functional lens.  Ultimately, my hope is that some functional concepts will bleed their way into my Ruby programming.

###Learning Clojure
I spent my first week at Hacker School learning the ins-and-outs of Clojure.  Why Clojure?  1) It has a (relatively) small API to learn, and 2) there are a suprising number of "Clojurists" at Hacker School. "Do things at Hacker School that would be a lot harder somewhere else," was one of the first bits of advice given to me on Day 0 of Hacker School.  Having a room full of Clojure experts to pair with flattened the initial learning curve considerably.

###String Compression
Learning to program functionally can be brain bending at first, so I chose small programming challenges to get started.  Here's my second program, which takes a string input from a user and compresses it by removing duplicate characters (eg, turn "aaaaaabbbcccdd" into "6a4b3c2d").     

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
  a string that replaces repeat characters with counts, eg 'a4b2c2d2e4f2'"
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

###Forgetting State
Right off the bat, I really liked letting my brain "forget" about state when composing the program.  In object oriented programming, objects encapsualte both state and methods. 

When programming in an object oriented paradigm, instantiated objects encapsulate methods AND state.    
One of the things I like right off the bat was freeing my brain from keeping track of sta 

