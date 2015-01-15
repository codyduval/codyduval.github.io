---
layout:     post
title:      "DCI Flavored Rails, Part 2"
date:       2015-01-10
summary:    "Part 2: The implementation, and its pros & cons."
categories: rails
---
Head over to [Part 1]({% post_url 2015-01-03-dci-rails-part-1 %}) for a summary of the core concepts behind the Data, Context, & Interaction architectural pattern.  Below you'll find how I implemented DCI in Rails, and the pros and cons that come with it.

###Start with a Use Case
For this experiment, we'll build an application to manage sign-ups at a play space for children.  In DCI, Contexts encapsulate a single use case.  We'll structure our first DCI context around the following use case:

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

Before we dive into the details of the Context, let's look at its public API. We access the Context through a Rails controller, so we'll start there.

###A Logic-less Controller
Since this is an exercise in "pure" DCI, we'll apply some strict rules to the rest of the app.  Our controllers' only concern will be  managing HTTP requests - no business logic or database access.   

{% highlight ruby %}
  class SessionRegistrationController < ApplicationController

    def home
      @home = SessionRegistering.start(current_user_id: current_user.id)
    end

    def register
      play_session = SessionRegistering.register(child_id: params[:child_id],
                                                 play_session_id: params[:play_session_id])
      if play_session
        #yay - show a success flash message
      else
        #boo - show some sort of flash error
      end
    end
  end
{% endhighlight %}

In accordance with Jim Gay's style (a Ruby/DCI proponent and author of Clean Ruby), we'll use a gerund to name our context class and call it `SessionRegistering`. Another common convention is to append "Context" to a use case name (eg `SessionRegisterContext`).

`SessionRegistering` has two public class-level methods.  These are our Context *triggers* in DCI parlance:

* `SessionRegistering.start` returns a hash with all the information Rails needs to build the initial view. 
* `SessionRegistering.register` takes the `POST` request on submit and returns the play session the parent just booked.

With that, let's dive into the `SessionRegistering` class itself.

###Casting Actors at Runtime
DCI has a few conditions that can be tricky to implement in Ruby.  The first is that our general domain Data objects (for example `User`), don't contain any methods specific to a individual context. As proof of that, here's our `User` class:  

{% highlight ruby %}
  class User < ActiveRecord::Base
  end
{% endhighlight %}

Yup, it's empty.  As of right now, all `User` behavior is specific to our one use case, encapsulated in our `SessionRegistering` class. Let's look at `SessionRegistering.start`:
    
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
    
    def start
      SessionRegisteringHomePresenter.new(parent, parent.kids, open_semesters, sessions).home
    end

    #...more code ..#
end
{% endhighlight %}

Setting aside the gnarly `#initialize` method for a second, lets look at how we are 'casting' behavior onto our domain objects when we call `SessionRegistering.new`. For a parent:

{% highlight ruby %}
  @parent = find_person(current_user_id).extend(Parent)
{% endhighlight %}

Since we're playing by strict DCI rules, all user behaviors (including associations!) are cast inside the context class.  We're doing this with Ruby's `Object#extend`, which adds behavior onto an object at runtime.  In the case of our parent, we extend the `Parent` module onto the more general `User` object. Here's that module:

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

After extending `User` with the `Parent` module, `User` obtains the ability to query for its children via the `#kids` method. You'll see that we're adding a new association onto the `User` class.  I know, pretty ugly, right?

One limitation of using `Object#extend` is that the new object is decorated for the life of the object - that is, there's no way to `unextend` it at the conclusion of the Context.  While this technically violates a "rule" of DCI, in practice it's not a huge deal in Rails as objects have relatively short life cycles.

###Roles Encapsulate Interactions
If you look at the full DCI Context class below, it's pretty easy to divine exactly what's going on.  Instead of overloading the `User` class with methods that apply to both kids and parents, we encapsulate their interactions within a Role that is applied dynamically to the base class.   

###The Full DCI Context

Here's the complete DCI context class.

{% highlight ruby %}

  # Primary Actor: a regular user cast as a parent
  # Goal: parent registers their child for an open play sesssion
  # Supporting Actors: the parent's child, a play session
  # Preconditions: parent is already authenticated and registered

  class SessionRegistering
    attr_accessor :parent, :child, :semester, :play_session

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
    
    def start
      SessionRegisteringHomePresenter.new(parent, parent.kids, open_semesters, sessions).home
    end

    def self.register(child_id, play_session_id)
      SessionRegistering.new(child_id: child_id, play_session_id: play_session_id).register
    end

    def register
      @play_session.add(child)
      if @play_session.save
        @play_session
      else
        false
      end
    end

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

    module Parent
      def kids
        self.class.class_eval do
          has_many :children, class_name: "User", foreign_key: "parent_id"
        end
        self.children
      end
    end

    module SelectedSemester
      def open_sessions
        self.play_sessions
      end

      def play_sessions
        self.class.class_eval do
          has_many :play_sessions
        end
      end
    end

    module OpenPlaySession
      def add(child)
        self.children << child
      end

      def kids
        self.class.class_eval do
          has_and_belongs_to_many :users
        end
        self.children
      end
    end

    def find_semester(semester_id)
      Semester.find(semester_id)
    end

    def find_person(user_id)
      User.find(user_id)
    end

    def find_play_session(play_session_id)
      PlaySession.find(play_session_id)
    end

    def self.open_semesters
      Semester.where(:open_for_reg => true)
    end

    def find_registration
      Registration.find(registration_id)
    end

  end
{% endhighlight %}

We'll skip over the remainder of the class details for now to focus on what matters: Does DCI lead to more readable and maintainable code? Is it a Good Thing™?

###Performance Issues
Setting aside any architectural opinions for a moment, there are real performance issues with using Ruby's `#extend` to add behavior to an object at runtime.  By calling `#extend` on an object, you obliterate Ruby's method cache for that object, making method lookup *an order of magnitude* slower (you can see some data [here](http://tonyarcieri.com/dci-in-ruby-is-completely-broken)).  I haven't done any benchmarking of my own, but I'm not sure how much this would matter in the context of a Rails app where IO is by far the biggest bottleneck.

Ruby 2.0 brought with it another way to add behavior to an object at runtime, via `UnboundMethod` (used in the [casting gem](https://github.com/saturnflyer/casting)). `UnboundMethod` has performance issues of its own, but it does provide a way to "unbind" functionality at the end of Context. 

Other techniques to decorate an object include using `SimpleDelegator` or `Forwardable` at the class level.  But using these wrapper methods violate "strict" DCI as you are not directly manipulating Data objects at runtime.

###Readability & Maintainability
If implemented correctly, DCI should make it easier to maintain and understand an application's code.  Rails is built around the [Active Record](http://www.martinfowler.com/eaaCatalog/activeRecord.html) pattern and while that affords some conveniences, it has the unfortunate side effect of abstracting away what the application actually *does*.  By placing business logic in its own class, it's much easier to reason on the actual functionality of the app. And since business logic is separated from domain logic, changes to the code base should be easier, too.     

###Not Very DRY
We've only implemented one Context, but what happens when another use case needs similar functionality to another?  How do we share functionality between Context classes?

DCI has a concept called (perhaps confusingly) "habits", that are meant to contain recurring use case fragments that don't have a specific goal.  In my own implementation, I think moving model associations into the Context was a step too far - those will almost certainly be reused. I don't have any validations here, but you could argue that those are specific to the Context but would also be reused elsewhere. 

###A Single Responsibility?
One tenet of good object oriented design is the *Single Responsibility Principle*. SRP states that a class should have one, and only one, reason to change.  In our experiment, we're squeezing a lot of behavior into 100+ line class.  But is it a *single* responsibility?  It is at a conceptual level, but in practice it's doing too many things.  The biggest clue here are all the default `nils` in the `#initialization` method.  When we initialize a new `SessionRegistering` class to render the page at start, we need a different set of data compared to when we save it.

If I were to refactor this experiment, I'd likely split these actions apart so that the `SessionRegistering` class is really only responsible for the `register` method.

###Summary
I really like the core ideas behind DCI, and I think some flavor of it makes a lot of sense in a Rails application.  This might mean straying slightly from the "pure" DCI path and employing service objects to encapsulate complex business logic.     

