---
layout:     post
title:      "DCI Flavored Rails, Part 1"
date:       2015-01-03
summary:    "In Part 1, DCI in a Nutshell"
categories: rails
---
DCI (or Data, Context, and Interaction) is a proposed compliment to the widely adopted Model-View-Controller architectural pattern. DCI was invented by [Trygve Reenskaug](http://en.wikipedia.org/wiki/Trygve_Reenskaug), a Norwegian computer scientist who also just happens to be the father/author of MVC. Around 2011/2012, DCI received a fair amount of attention within the Ruby and Rails community as an antidote (or perhaps just an alternative) to the less-than-ideal "Fat Model / Skinny Controller" advice that was en vogue at the time.

More recently, Jim Gay released his [Clean Ruby](http://clean-ruby.com/) book which goes into some detail on implementing DCI within Ruby.  After reading Clean Ruby, I was inspired to try DCI within the context of a Rails application.  As with anything in programming, there are always trade-offs.  Read on for a summary of DCI, or skip to [Part 2]() to see how I implemented it in Rails.

###Why DCI?
DCI has a number of goals, but at its core it aims to separate *being* from *doing* in a program.  Said another way, DCI provides an architecture for separating what a system is (the domain - which doesn't change very often) from what it does (the business processes and logic -  which are likely to change a lot over the lifetime of the app). Trygve Reenskaug probably said it best:   

>Object-oriented programming was supposed to unify the perspectives of the programmer and the end user in computer code: a boon both to usability and program comprehension. While objects capture structure well, they fail to capture system action. DCI is a vision to capture the end user cognitive model of roles and interactions between them.

If you've worked in a long-lived Rails app, you've likely experienced huge "god" classes (often a `User`) that inherit from ActiveRecord and contain business logic, validations, and knowledge about the entire system.  Any change to a god class is usually painful, and just trying to reason what the system does can require reading hundreds of lines of code.

By giving system behavior first-class status, DCI aims to produce programs that are easier to maintain and reason about.

###DCI as a Play (A Leaky Analogy)
DCI is sometimes explained by comparing it to a play (as in, a theater production).  It's not a perfect metaphor, but it's a useful starting point.

D is for **Data** and it represents the (relatively) static data model of an object in our domain.  This is where we represent the user's mental model of *things* within our system.  The trick here is to define the interface to a Data object with enough detail to capture the universal domain properties. We should leave out anything that is unique to a specific scenario or use case. In a Rails app, Data is usually represented by our ActiveRecord objects. And in our play analogy, **a yet-to-be cast actor** could represent a data object.

I is for **Interaction,** which represents the algorithms of how our Data objects interact with one another.  Interactions are defined in terms of *Roles*, which contain behavior for a specific scenario.  In our play, actors (Data) are cast into roles.  A Role comes with lines, stage directions, a costume (etc. etc.) - all specific interactions for a specific scene in our play. It's important to note that once a Data object is finished with its role in the Context (see below), it is stripped (or uncast) of any context specific behavior.

Finally, C is for **Context,** which encapsulates the business logic of our application. Something is *happening* inside a context, and you should be able to divine what an application does by looking inside its various Contexts.  Within our play analogy, the instructions a director gives to her actors for a specific scene most closely mirrors a DCI Context.  A director *casts* actors into specific roles(which contain information about how to interact with other actors cast into roles), tells them when to enter and exit the scene, and so on.  Often, there's a one-to-one relationship between a use case and a Context. In a Rails app, you should be able to open a Context class and quickly reason what the program is doing.

In DCI, we are most concerned with the *Interactions* between data objects.  Those *Data* objects are cast as roles depending on the *Context* and specific business use case (more on that later).

With that, read on to [Part 2]() to see my implementation of DCI inside Rails, and the pros/cons it brings.
