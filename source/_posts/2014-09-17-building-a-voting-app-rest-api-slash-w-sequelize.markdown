---
layout: post
title: "Building a Voting App: REST API /w Sequelize"
date: 2014-09-17 12:15:56 +0300
comments: true
categories: nodejs REST Sequelize backend
tags: [nodejs, REST, Sequelize, backend]
comments: true
share: true
image:
  feature: /images/diary-bg20.jpg
---

## Introduction

[Sequelize](http://sequelizejs.com) is an ORM (Object Relational Mapper) for NodeJS that provides easy access to various databases like MySQL, MariaDB, SQLite, or PostgreSQL.

It has a nice website with helpful documentation and [examples](https://github.com/sequelize/sequelize/tree/master/examples) to get started. Also there is an IRC channel #sequelizejs on Freenode.

In this article we will build a voting application where users can vote for a poll and we will record their `IP` so they cannot vote twice. We will use `sequelize` to model our data and build a `REST API`. Later we will build the front end using `emberjs`.

## Scaffold Voting Application

Since we are using `emberjs` we might use `ember-cli` for our project. But, yet recently I published a yeoman generator [generator-emberfs](https://github.com/eguneys/generator-emberfs) for emberjs fullstack projects like the one we are building now. It does all the scaffolding for emberjs on the client side and expressjs on the server side. So we will use it for this article.

You can install using the command:

{% highlight bash %}
  npm install -g generator-emberfs
  yo emberfs
{% endhighlight %}

### Directory Structure

This is our directory structure for voting application. `config` folder contains the initial server code and configuration files. `app/models`, `app/routes` and `app/views` folders contain our server side models, routes, and views respectively. `db` folder is for the sqlite database file and code for our database seed.

{% highlight bash %}
.
|-- app
|   |-- models
|   |   |-- index.js
|   |   `-- poll.js
|   |-- routes
|   |-- `-- api.js
|   `-- views
|       `-- layouts
|-- config
|   |-- app.js
|   `-- server.js
|
`-- db
    |-- development.sqlite
    `-- seed.js
{% endhighlight %}

We will define only routes for the REST API so we won't mention about the `app/views` folder here.

Note that this folder structure is boilerplate for the `generator-emberfs`. So you won't need to do the tedious work if you use it.

## Models in Sequelize

This is our model diagram. *Poll* is a *question* and has many *Choices*. *Choice* is a poll choice with a *text* and *description* and has many *Votes*. *Vote* has an *ip* that identifies the voter, so we can make sure one ip can vote only once.

[image](vote_diagram.png)

All our models are in file `app/models/poll.js`. Later we will import them.

**Vote** model is simple, just a single field `ip` and has no relationships. It also has some helpers methods which I will mention later.

{% highlight js %}

  var Vote = sequelize.define('Vote', {
    ip: DataTypes.STRING
    // ... define model fields here
  }, {
    classMethods: {
      associate: function(models) {
        // define relationships here
      }
        // ... define class methods here
    }
  });

{% endhighlight %}

**Choice** model has two fields, `text` and `description`, and has one-to-many relationship with the *Vote* model. We define the relationships in a class method named *associate*.

{% highlight js %}

  var Choice = sequelize.define('Choice', {
    text: DataTypes.STRING,
    description: DataTypes.STRING
  }, {
    classMethods: {
      associate: function(models) {
        Choice.hasMany(models.Vote, { as : 'votes' });
      }
    }
  });

{% endhighlight %}

**Poll** model has one field *question*, and has one-to-many relationship with the *Choice* model.

{% highlight js %}
  var Poll = sequelize.define('Poll', {
    question: DataTypes.STRING
  }, {
    classMethods: {
      associate: function(models) {
        Poll.hasMany(models.Choice, { as: 'choices' });
      }
    }
  });
{% endhighlight %}

### Import all models in Sequelize

Now we have defined our models, we need to import them with sequelize. We will do that in file `app/models/index.js`.

In our example we have only one file defining our models namely `app/models/poll.js`. But we can have more than that, and we will import all the files in `app/models` folder.

First let's require our dependencies, and initialize Sequelize. `votes-app-db` is the name of our database.

We are using `sqlite` for the database and `./db/development.sqlite` file for the database. `sqlite` is a zero-configuration, serverless database. It works directly from the database files on disk.

`File: app/models/index.js`

{% highlight js %}

var fs = require('fs'),
  path = require('path'),
  lodash = require('lodash'),
  Sequelize = require('sequelize'),
  sequelize = null,
  db = {};

sequelize = new Sequelize('votes-app-db', null, null, {
  dialect: 'sqlite',
  storage: './db/development.sqlite'
});

{% endhighlight %}

Before we import the models, let's see how we export our models in `app/models/poll.js`.

`File: app/models/poll.js`

{% highlight js %}

var Sequelize = require('sequelize');

module.exports = function(sequelize, DataTypes) {
  // I omit the definitions.
  var Vote, Poll, Choice;

  return [Vote, Poll. Choice];
};

{% endhighlight %}

We export an array of our models. Now let's import each file in `app/models` to sequelize.

`File: app/models/index.js`

{% highlight js %}

fs.readdirSync(__dirname)
  .filter(function(file) {
    return (file.indexOf('.') !== 0) && (file !== 'index.js');
  })
  .forEach(function(file) {
    var model = sequelize.import(path.join(__dirname, file));
    if (model instanceof Array) {
      model.forEach(function(m) {
        db[m.name] = m;
      });
    } else {
      db[model.name] = model;
    }
  });

{% endhighlight %}

We filter out the `index.js` and import every file to sequelize. Later we keep references to our models in `db` object. Note that our model files can both return an array or a single object, that's what the *if* condition is checking for.

Finally we call the `associate` class method for each model that will define the relationships between models. Then we export all the models (`db` object) along with sequelize.

{% highlight js %}

Object.keys(db).forEach(function(modelName) {
  if ('associate' in db[modelName]) {
    db[modelName].associate(db);
  }
});

module.exports = lodash.extend({
  sequelize: sequelize,
  Sequelize: Sequelize
}, db);

{% endhighlight %}

### Seed for the Database

Next, we seed the database a sample poll with two choices.

`File: db/seed.js`

{% highlight js %}

module.exports = function(db) {

  db.Poll.create({
    question: 'What features would you like to see'
  }).success(function(poll) {
    db.Choice.bulkCreate([{
      id: 1,
      text: 'Improved UI',
      description: 'New, fancy interface',
      PollId: poll.id
    }, {
      id: 2,
      text: 'Improved Performance',
      description: 'Faster response times',
      PollId: poll.id
    }]);
  });

};

{% endhighlight %}

Note that each model has a `create` and `bulkCreate` function. The difference is `create` takes a single object and `bulkCreate` takes an array of objects.

One final thing is the *choice* model instances has an extra attribute, *PollId*, that is the foreign key referencing the *Poll* model.

### Helper methods for Models

We will define two helper methods for the *Vote* model. They are both defined under the *classMethods* object (near to *associate* method).

**findCount** method takes a *choiceId* and finds the number of votes for that choice.

`File: app/models/poll.js`

{% highlight js %}

  findCount: function(choiceId, callback) {
    Vote.count({
      where: { 'ChoiceId': choiceId}
    }).success(function(count) {
      callback(null, count);
    });
  }

{% endhighlight %}

**addVote** method takes an *ip* and a *choice* and creates a new vote. The trick here is that we have to make sure the given ip doesn't vote twice for the same poll. So we need a nifty sql statement to "select all the votes for the poll that the given choice belongs to and that is the same ip as the given ip". This statement will return a single vote if the given ip has voted for the same poll or null if it hasn't.

`File: app/models/poll.js`

{% highlight js %}

  addVote: function(ip, choice, callback) {
    // sequelize translation of the sql command:
    //
    // select * from votes, choices, polls where
    // votes.choiceId == choices.id and choices.pollId ==
    // polls.id and choices.id == choice polls.id in
    // (select pollId from votes, choices where
    // votes.choiceId == choices.id and votes.ip == ip)
                
    Poll.findAll({ where: Sequelize.and(
      {
        'choices.id': choice
      },
      {id: 
        { in:
          Sequelize.literal('select "Choices"."PollId" from "Votes", "Choices" where "Votes"."ChoiceId" = "Choices"."id" and "Votes"."ip" = \'' + ip + '\'')
        }
      }),
        include: [ {
          model: Choice, as: 'choices',
          include: [{ model: Vote, as: 'votes' }]}]
      }).success(function(votes) {
        if (!votes || votes.length === 0) {
          Vote.create({
            ip: ip,
            ChoiceId: choice
          })
            .success(function(vote) {
              callback(null, vote);
            });
        } else {
          callback("Vote already exists");
        }
      });
  },
            
{% endhighlight %}

I am sure there is a better way to build this statement in sequelize, but this works too. Note that we make use of `Sequelize.literal` method that interprets the raw sql command.

Finally upon success, if the vote already exists, we give error, otherwise we create a new vote which builds and saves the vote to the database.

## Routes for Express 4

Routing in Express 4 is modular and mountable. A `Router` instance is a complete middleware. But, yet I won't get into detail how to use express routers in this article. For more information check out the express [docs](http://expressjs.com) and the [migration guide](http://expressjs.com/migrating-4.html#routing).

All our API routes are in file `app/routes/api.js`.

`/polls` route lists all the available polls.

{% highlight js %}
    
  router.get('/polls', function(req, res) {
    db.Poll.findAll({
      // you can include choices for the poll, or not.
      //include: [{ model: db.Choice, as: 'choices' }]
    }).success(function(polls) {
      res.send({
        polls: polls
      });
    });
  });

{% endhighlight %}

Note that `include` option enables us to embed the *choices* for the poll. In our case we opt not to.

`/polls/:id` route gets a single poll with a given id. This time we *include* the choices for the poll as well. The trick here is for each choice we find the number of votes for that choice, and include the count in the choice model using the `setDataValue` method.

{% highlight js %}

  router.get('/polls/:id', function(req, res) {

    db.Poll.find({
      where: { id: req.params.id },
      include: [{model: db.Choice, as: 'choices' }]
    }).success(function(poll) {
      async.map(poll.choices, function(choice, callback) {
        db.Vote.findCount(choice.id, function(err, count) {
          choice.setDataValue('count', count);
          callback(err, choice);
        });
      }, function(err, results) {
        poll.setDataValue('choices', results);
        res.send({
          poll: poll
        });
      });
    });
  });

{% endhighlight %}

Note that we use *async* library to map *poll.choices* and add the vote count for each choice. Later we use `setDataValue` method for the poll model to set the *choices* to the map results.

We post to `/votes` route to make a vote. One ip can vote for a poll only once, thanks to our *addVote* helper method.

{% highlight js %}

  router.post('/votes', function(req, res) {
    var ip = req.headers['x-forwarded-for'] || req.connection.remoteAddress;
    // randomize the ip to test your app
    //ip = Math.random();
        
    var choice = req.body.vote.choice;
        
    db.Vote.addVote(ip, choice, function(err, vote) {
      if (err) {
        res.send(err);
      } else {
        res.send(vote);
      }
    });
  });

{% endhighlight %}

Note that we use `x-forwarded-for` header or the `req.connection.remoteAddress` to get the ip. If you want to test the application so you can have many votes, randomize the ip with `Math.random`.

Finally `/votes/:choiceId` route will return the number of votes for a given choice.

{% highlight js %}

  router.get('/votes/:choiceId', function(req, res) {
    var choice = req.params.choiceId;

    db.Vote.findCount(choice, function(err, count) {
      if (err) {
        res.send(err);
      } else {
        res.send({ vote: count });
      }
    });
  });

{% endhighlight %}

## Setup the Server

Finally we start sequelize, seed the database, and fire up the express server.

`File: app/config/app.js`

{% highlight js %}

var app = require('./server'),
  db = require('../app/models');

db.sequelize
  .sync({ force: true})
  .complete(function(err) {
    if (err) {
      throw err[0];
    } else {
      // seed
      require('../db/seed')(db);
      
      app.listen(app.get('port'), function() {
        console.log('express listening on ' + app.get('port'));
      });
    }
  });
    
{% endhighlight %}

Note that we require `../app/models` that exports all the *models* and *sequelize* as the *db* object. Also we require `./server` that exports the express application. I won't mention about configuring and setting up the express server, it's simple enough and you should do it however you like. For more details check out the full repository at [github](https://www.github.com/eguneys/voting_app).

## Conclusion

In this article we have built a `REST API` using `sequelize` for voting polls. We've setup models in `sequelize` provided seed for the database, used query methods for the models, and setup the API routes for the `express` server. Next up we will build the front end using `emberjs`.

The full repository is at [github](https://www.github.com/eguneys/voting_app).
