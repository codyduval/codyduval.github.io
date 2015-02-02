---
layout:     post
title:      "One Way Data Binding is the New Black"
date:       2015-01-25
summary:    ""
categories: react
---
Building responsive user interfaces for the web is hard. More specifically, keeping track of a UI's mutable state is hard. A single button click may trigger a cascade of changes throughout the view.  Keeping track of (and synchronizing) these changes can be very challenging in complex interfaces.  Prior to React's emergence a few years ago, observers and two-way data binding were our best solution to this tough problem.  React has shown that what's old is new again. One-way data binding is the new black. 

I built a (relatively) simple UI using React to explore the pros (and cons) of React's uber features: immutablity, the virtual DOM, and unidirectional data flow.  I really enjoyed the process and I'm genuinely excited about building responsive UIs with React.

###Modeling Mutable State
In the old days, we built "responsive" interfaces with technologies like Flash, CGI, or DHTML. Today, we'd use a big 'ole javascript framework like Ember or Angular.  Most modern javascript MVC frameworks use two-way data binding to manage appliction state: changes to the view update the data model, and changes to the data model update the view.  This works fine in simple applications, but it quickly becomes tangled in more complex (and long lived) programs.  Changes in one data model can trigger a change in another model, and it's easy to fall into the trap of using two-way bindings as a communication channel between components.

That's all well and good, but what's the alternative?  React's "big idea" was to make most of an application's state in the view immutable. So instead of updating and observing state for individual DOM nodes, React demands that a single parent component (ie a DOM node) contains all of the data for its sub children.

>  React components are basically idempotent functions - they describe your UI at any point in time, just like a server-rendered app.
-- Pete Hunt, React.js maintainer

 So when state changes, React re-renders the *entire DOM* and eliminates the need to reason about state change over time.  More on that in a second, for now, let's look at some code:

{% highlight javascript linenos %}
/** @jsx React.DOM */

var Registration = React.createClass({

  handleSelect: function(sessionName) {
    this.setState({ selectedSession: sessionName });
  },

  getInitialState: function () {
    return { sessions: this.props.initialSessions,
             user: this.props.user,
             selectedSession: "",
             kids: this.props.kids };
  },

  render: function () {
    var kid;
    if (this.props.kids.length > 1) {
      kid = _.first(this.props.kids);
    } else { 
      kid = _.first(this.props.kids);
    }
    
    var regBox;
    if (this.state.selectedSession !== "") {
      regBox = <RegistrationsBox user={this.state.user} selectedSession={this.state.selectedSession} kids={kid}  />
    }

    return (
      <div className="container">
        <div className="col-md-8">
          <SessionsBox initialSessions={this.state.sessions} selectedSession={this.state.selectedSession} onSelect={this.handleSelect} />
        </div>
        <div className="col-md-6 col-md-offset-2">
          {regBox}
        </div>
      </div>
      );
  }
});

{% endhighlight %}

This post isn't meant to be a full tutorial on React - we'll just focus on a few key concepts.  You'll notice on lines 9 - 13 that this component is setting its initial state via `props`, which are immutable properties passed into the object.  Lower down, on line 26, you'll see that a `RegistrationsBox` is rendered as a child of the `Registration` component, and its own properties (`user` and `selectedSession`) are set via `this.state` calls.  Unlike `props`, `state` is meant to contain things that change - in this case, the selected user and the session they have selected to sign up for.

Next, if we dive into the `RegistrationsBox` component and it's own child `RegistrationsList`:

{% highlight javascript linenos %}
/** @jsx React.DOM */

var RegistrationsBox = React.createClass({

  render: function () {
    return (
      <div className="well">
        <table className="table">
          <RegistrationsList selectedSession={this.props.selectedSession} kids={this.props.kids} />
        </table>
      </div>
      );
  }
});

var RegistrationsList = React.createClass({
  render: function () {
    return (
      <div>
        <thead>
          <tr>
            <td><h2>Selected Session</h2></td>
            <td><h2>Child</h2></td>
            <td><h2>Price</h2></td>
            <td></td>
          </tr>
        </thead>
        <tbody className="registration-list">
          <tr>
            <td>
              <div className="session-name">
                {this.props.selectedSession}
              </div>
            </td>
            <td>
              <div className="session-child">
                {this.props.kids.first_name}
              </div>
            </td>
            <td>
              <div className="session-price">
                $45 
              </div>
            </td>
            <td>
              <div className="session-submit">
                <button onClick={this.handleSubmit} className={'btn btn-success'}>Check Out</button>
              </div>
            </td>
          </tr>
        </tbody>
      </div>
    );
  }
});

{% endhighlight %}

You'll see the `selectedSession` and `kids` data "rolling downhill" and pass through the `RegistrationsBox` component and into the `RegistrationsList`. Once there, the actual values are displayed within the JSX.  The important point here is that if any of these values change (due to a button click, or whatever), the entire tree will be re-rendered in React's virtual DOM.  Then, some fancy algorithims will compare the virtual DOM to the actual DOM, and force an update in a single batch.

The completed UI is posted [on this gist]().  I've had a ton of fun playing with React and hope to use it more in the future.




