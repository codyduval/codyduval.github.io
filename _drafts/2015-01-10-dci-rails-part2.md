---
layout:     post
title:      "DCI in Rails, Part 2"
date:       2015-01-10
summary:    "The pros & cons of DCI inside Rails"
categories: rails
---
Head over to [Part 1]({% post_url 2015-01-03-dci-rails-part-1 %}) for a summary of the core concepts behind the Data, Context, & Interaction architectural pattern.  Below is how I implemented DCI in Rails, and the pros and cons that come with it.  

###Start with a Use Case
For this experiment, we'll build an application to manage sign-ups at a play space.  In DCI, Contexts encapsulate a single use case.  For our app, we'll build our first DCI context around the following use case:

**Name:** Register Child for a Play Session
**Summary:** A parent wants to sign their child up for a recurring session at a local play space. The system will show the parent the available play sessions for the current or future semesters, and the parent will be able to add their child to a single open session.
**Preconditions:** A parent has previously signed up for an account in the system and is logged in. 
**Description (Sunny Day Scenario):** 
1. System presents parent list of open play sessions (organized by semester).
2. Parent selects session
3. Parent assigns child to session (unless parent only has one child, in which case this happens automatically).
4. Parent confirms selection and is taken to checkout/payment flow.
**Exceptions (Rainy Day Scenario):**
* There are no open play sessions.
* Parent's payment fails.
**Postconditions:** On successful payment, system assigns child to play session and decreases open slots in session by one.

This use-case nicely encapsulates a single DCI Context.  Our Context will consist of a single Ruby class, and this class will instantiate all of the objects needed to successfully complete the use case.   
Before we dive into the details of let's look at it's public API which we access through a Rails controller.

###A Logic-less Controller
Since this is an exercise in "pure" DCI, we'll apply some strict rules to the rest of the app.  Our controllers only concern will be  managing HTTP requests - no business logic or database access.   

{% highlight ruby %}
  class SessionRegistrationController < ApplicationController

    def home
      @home = SessionRegistering.start(current_user.id)
    end

    def register
      play_session = SessionRegistering.register(child_id: params[:child_id], play_session_id: params[:play_session_id])
      if play_session
        #yay - show a success flash message
      else
        #boo - show some sort of flash error
      end
    end
  end
{% endhighlight %}

In accordance with Jim Gay's style (a Ruby/DCI proponent and author of Clean Ruby), we're using gerunds to name our context classes and our use case is encapsulated in a `SessionRegistering` context.  `SessionRegistering` has two public class methods: `SessionRegistering.start` and `SessionRegistering.register`. (We're using class level methods for readability and brevity, but these could easily be instance methods, too.)

`SessionRegistering.start` returns a hash with all the information Rails needs to build the initial view. And `SessionRegistering.register` takes the `POST` request on submit and returns the play session the parent just booked.

With that, let's dive into the `SessionRegistering` class itself.

###Casting Actors at Runtime
DCI has a few conditions that can be tricky to implement in Ruby.  The first is that our general domain objects (for example `User`), don't contain any methods specific to a individual context.  Let's look at `SessionRegistering.start`:
    
{% highlight ruby %}
  class SessionRegistering
  
    def initialize(current_user_id: nil,semester_id:  nil,
                   child_user_id: nil, play_session_id:  nil,
                   reg_id:  nil)
      @child = find_person(child_user_id).extend(Child)
      @parent = find_person(current_user_id).extend(Parent)
      @semester = find_semester(semester_id).extend(SelectedSemester)
      @play_session = find_play_session(play_session_id).extend(OpenPlaySession)
    end

    def self.start(current_user_id)
      SessionRegistering.new(current_user_id: current_user_id).start
    end
    
    def start(current_user_id)
      SessionRegisteringHomePresenter.new(@parent, @parent.kids, open_semesters, sessions).home
    end

    #...more code ..#
end
{% endhighlight %}

Setting aside the gnarly `#initialize` method for a second, lets look at how we are 'casting' behavior onto our domain objects within the Context class.  Our domain object `User`, is simply an empty class that inherits from `ActiveRecord`: 

{% highlight ruby %}
  class User < ActiveRecord::Base
  end
{% endhighlight %}

Since we're playing by strict DCI rules, all user behaviors (including associations!) are cast inside the context.  We're doing this with Ruby's `Object#extend`, which adds behavior onto an object at run time.  In the case of our parent, we extend the `Parent` module onto the more general `User` object.    

{% highlight ruby %}
  class SessionRegistering

    #... code ...#

    module Parent
      def kids
        self.class.class_eval do
          has_many :children, class_name: "User", foreign_key: "parent_id"
        end
        self.children
      end

    #... more code ...#

    end
{% endhighlight %}

Now, `User` now has a `#kids` method which is simply adding a new association onto the (empty) `User` class.  I know, pretty ugly, right?  Remember, this is an experiment, so we'll tally up the pros and cons later.

Similarly, we can `#extend` the `Child` module onto `User` to give the `User` object kid specific methods.

{% highlight ruby %}
  class SessionRegistering

    #... code ...#
      module Child
        def parent 
          self.class.class_eval do
            belongs_to :parent, :class_name => "User", :foreign_key => :id, :primary_key => :parent_id
          end
          self.parent
        end

        def play_sessions
          self.class.class_eval do
            has_and_belongs_to_many :play_sessions
          end
          self.play_sessions
        end
      end

    #... more code ...#

    end
{% endhighlight %}

Is this a good thing?  Setting aside any architectural opinions for a moment, there are real performance issues with using Ruby's `#extend` to add behavior to an object at runtime.  By calling `#extend`, you obliterate Ruby's method cache, making method lookup *an order of magnitude* slower (you can see some data [here](http://tonyarcieri.com/dci-in-ruby-is-completely-broken)).  I haven't done any benchmarking of my own, but I'm not sure how much this would matter in the context of a Rails app where IO is by far the biggest bottleneck.

Ruby 2.0 brought with it some other ways of adding behavior to an object at runtime, including `UnboundMethod` (used in the [casting gem](https://github.com/saturnflyer/casting)). `UnboundMethod` doesn't destroy the method cache, but has performance issues of its own.


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
