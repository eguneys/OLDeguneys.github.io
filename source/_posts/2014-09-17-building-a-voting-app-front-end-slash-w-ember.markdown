---
layout: post
title: "Building a Voting App: Front-end /w ember"
date: 2014-09-17 12:16:34 +0300
comments: true
categories: nodejs ember frontend
tags: [nodejs, ember generator-emberfs ember-cli frontend]
comments: true
share: true
image:
  feature: /images/diary-bg20.jpg
---

## Introduction

In the [previous article](/blog/2014/09/17/building-a-voting-app-rest-api-slash-w-sequelize/) we've built a REST API for a voting application using sequelize. In this article we will consume this API and build the front end using emberjs. We also mentioned about, [generator-emberfs](https://github.com/eguneys/generator-emberfs), a yeoman generator for fullstack applications using emberjs and express, that we used to scaffold our application.

The voting application will be where users can vote for a poll and see the results of the poll. You can see a live example [here](http://votefree.herokuapp.com).

## Scaffold Ember Application

In the previous article we used generator-emberfs for server side, we will continue using it for client side as well. It supports scaffolding, asset compilation, optimization for production, live reloading and test driven development. It uses [gulp](http://gulpjs.com) for build system.

You can launch the server and listen for changes using the command:

{% highlight bash %}
  gulp devserver
{% endhighlight %}

Or you can build and optimize your application for production:

{% highlight bash %}
  gulp build
{% endhighlight %}

### Directory Structure

This is our directory structure for client side. `vendor` folder contains third party libraries, `templates` folder contains handlebars templates, `styles` folder contains css styles in sass, `scripts` folder contains the emberjs application code.

{% highlight bash %}

app/client
|-- scripts
|   |-- app.js
|   |-- common.js
|   |-- components
|   |   `-- progress-bar.js
|   |-- controllers
|   |   `-- index_controller.js
|   |-- main.js
|   |-- mixins
|   |-- models
|   |   `-- poll_model.js
|   |-- router.js
|   |-- routes
|   |   `-- index_route.js
|   `-- views
|-- styles
|   `-- style.scss
|-- templates
|   `-- components
`-- vendor

{% endhighlight %}

Note that this folder structure is boilerplate for the generator-emberfs. It uses requirejs with AMD modules. But, yet we won't get into details of requirejs in this article, rather stick to essential codes for emberjs.

## Routes in Ember

We won't define any particular route for this simple application. Rather we will use the default `IndexRoute` Ember provides.

{% highlight js %}
App.Router.map(function() {
});

App.router.reopen({
  location: 'history'
});
{% endhighlight %}

Our `IndexRoute` will have the poll #1 as its model. Remember we have provided a seed model as the poll #1 in sequelize, in the previous article.

{% highlight js %}
App.IndexRoute = Ember.Route.extend({
  model: function() {
    // GET api/v1/polls/1
    return this.store.find('poll', 1);
  }
});
{% endhighlight %}

`this.store` uses ember-data to pull the models. It uses an adapter, which is in our case a `DS.RESTAdapter`. So this code will make a request to `api/v1/polls/1` to build a model. Later we can use the model in our controller and templates.

### Templates in Handlebars

Let's drop a skeleton template using bootstrap for style and handlebars for bindings. I will use emblem formatting just for efficient view.

`File: templates/index.hbs`

{% gist b46c4bce03a532b48397 %}

Here, we have a `{{question}}` binding, main header for the poll. and one `div {{resultsHidden::hidden}}` binding for showing the poll choices and allowing vote. and a `div {{resultsHidden:hidden}}` binding for showing the poll results.

In the poll results we use a, `progress-bar text=text percent=vote-percent`, ember component to display the results, which we will mention later.

Finally we have various actions for communicating with the model `click="showResults"`, and `click="vote id"`. These actions are handled in ember controller and will be discussed later.

## Models in Ember Data

Just as we defined data models for sequelize, we will define the same models for emberjs using ember-data. We will use ember-data `DS.RESTAdapter` for communicating with the REST API.

In the previous article We mentioned about our three models `Poll`, `Choice`, and `Vote` and the relationships between them. Here we define the same relationships using ember-data.

`File: models/poll_model.js`

{% highlight js %}

  App.Poll = DS.Model.extend({
    question: DS.attr('string'),
    choices: DS.hasMany('choice')
  });

  App.Choice = DS.Model.extend({
    text: DS.attr('string'),
    description: DS.attr('string'),
    count: DS.attr('number'),
    votes: DS.hasMany('vote'),
    poll: DS.belongsTo('poll')
  });

  App.Vote = DS.Model.extend({
    choice: DS.belongsTo('choice')
  });
  
{% endhighlight %}

Note that ember uses `DS.attr` for defining attributes, and `DS.hasMany`, `DS.belongsTo` methods for defining relationships.

If you visit [/api/v1/polls/1](http://votefree.herokuapp.com/api/v1/polls/1) in your local server, you will see the json response for the poll #1:

{% highlight js %}
{"poll":{
  "id":1,
  "question":"What features would you like to see",
  "choices":[{
    "id":1,
    "text":"Improved UI",
    "description":"New, fancy interface",
    "PollId":1,
    "count":0},{
    "id":2,
    "text":"Improved Performance",
    "description":"Faster response times",
    "PollId":1,
    "count":0
    }]
}}
{% endhighlight %}

Notice `choices` attribute is embedded within the `poll` model. Normally ember-data expects this to be side loaded. So we will have to use `DS.EmbeddedRecordsMixin` to enable ember-data to parse our embedded response.

{% highlight js %}
App.PollSerializer = DS.RESTSerializer.extend(DS.EmbeddedRecordsMixin, {
  attrs: {
    choices: { embedded: 'always' }
  }
});
{% endhighlight %}

## Controllers in Ember

We will extend the default `IndexController` in our application. In ember, controllers communicate between the template and the model the extra properties defined in controller will be also available to the template. So ember controllers is in a way model decorators.

### Controller Computed Properties (Model Decorators)

`IndexController` extends `Ember.ObjectController` and decorates the `poll` model. We have various computed properties that will be available to our template, and they will be live bounded to our model.

`File scripts/controllers/index_controller.js`

{% highlight js %}

App.IndexController = Ember.ObjectController.extend({
  resultsHidden: true,
  
  pollChoices: function() {
    var choices = this.get('model.choices');
    
    var totalVotes = this.get('totalVotes');

    choices.map(function(item) {
      var votePercent = (item.get('count') / totalVotes) * 100;
      item.set('vote-percent', votePercent.toFixed(2));
      return item;
    });
    return choices;
  }.property('model.choices'),

  totalVotes: function() {
    var choices = this.get('model.choices');

    var totalVotes = choices.reduce(function(prev, item) {
      return prev + item.get('count');
    }, 0);

    return totalVotes;
  }.property('model.choices'),

  // ...
});

{% endhighlight %}

`resultsHidden` property toggles between poll results and voting. `pollChoices` computed property computes the total percentage of votes for each choice. `totalVotes` computed property computes the total number of votes for a poll. Note that `.property('model.choices')` function that is appended to computed property methods. That binds the `model.choices` to our computed property so whenever a change occurs computed property updates live.

### Controller Actions

Controller also communicates between template and model by means of actions. Remember we had three actions in our template, we handle those actions inside our controller.

`File: scripts/controllers/index_controller.js`

{% highlight js %}

  actions: {
    showResults: function() {
      this.get('model').reload().then(function() {
        this.set('resultsHidden', false);
      }.bind(this));
    },
    hideResults: function() {
      this.set('resultsHidden', true);
    },

    vote: function(choiceId) {
      var choices = this.get('model.choices');

      var vote = this.store.createRecord('vote', {
        choice: choices.findBy('id', '' + choiceId)
      });

      var self = this;

      vote.save().then(function(vote) {
        self.send('showResults');
      }, function(vote) {
        self.send('showResults');
      });
    }
  }

{% endhighlight %}

`showResults` action toggles the `resultsHidden` property to false so the results are shown. Note that before showing the results, it reloads the model so we get the latest results. `hideResults` is the opposite, it goes back to the voting view. `vote` action is where the user votes. It takes a `choiceId` parameter that is the poll choice. We create a new `vote` record and save it.

## Progress Bar Ember Component

Now let's see how we define an ember component. In our case a progress bar for displaying the percentage of votes for a poll choice.

First we define a template for our ember component.

`File: templates/components/progress-bar.hbs`

{% gist 248e17bdd390dbf5ec83 %}

Next we define the code for our ember component.

`File: scripts/components/progress-bar.js`

{% highlight js %}

App.ProgressBarComponent = Ember.Component.extend({
  'progress-bar-type': function() {
    var barTypes = [
      'progress-bar-danger',
      'progress-bar-warning',
      'progress-bar-info',
      'progress-bar-success'
    ];
    
    var percent = Math.min(this.get('percent'), 100);
    var barType = barTypes[Math.floor((percent - 1) / 25)];

    return barType;
  }.property('percent'),

  'percent-style': function() {
    return 'width:' + this.get('percent') + '%';
  }.property('percent')
});

{% endhighlight %}

Here, we calculate `progress-bar-type` based on percentage, that will apply the bootstrap classes to the div and make the progress bar change colors with the percentage.

## Setup the Ember Application

Finally we start our ember application and extend our ember application adapter.

{% highlight js %}
var App = Ember.Application.create();

App.ApplicationAdapter = DS.RESTAdapter.extend({
  namespace: 'api/v1'
});
{% endhighlight %}

Note that extend the `DS.RESTAdapter` and we provide a `namespace` option to tell it that our REST API resides at `api/v1` namespace.

## Development and Production

You can use `gulp test` to run your client side tests. generator-emberfs uses coffeescript for test code. You can use ember-qunit for unit testing and ember test-helpers for integration tests. It uses testem as the test runner. But, yet since we are using a server side REST API, You have to provide an [API Proxy](https://github.com/airportyh/testem#api-proxy) configuration to testem to redirect your API calls to your server, and launch your server before running the tests.

At this point we have finished the application and it's ready to ship. But first you need to build it using the command `gulp build`. This will compile all your assets, minify and optimize and concatenate your script and css files into the `public` directory. Now you can launch your server using the command: `node config/app`.

## Conclusion

In this article we have built a front end for a REST API where users can vote for a poll and see the results of the poll using emberjs. We've built a template in handlebars, setup the models in ember-data, extended an ember controller and an ember component for displaying progress bars.

The full repository is at [github](https://www.github.com/eguneys/voting_app). A live example is available [here](http://votefree.herokuapp.com)
