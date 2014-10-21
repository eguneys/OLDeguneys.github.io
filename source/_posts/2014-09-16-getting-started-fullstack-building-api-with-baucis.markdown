---
layout: post
title: "Getting Started Fullstack: Building API with Baucis"
date:   2014-01-06 09:52:06
categories: nodejs mongodb mongoose baucis
tags: [nodejs, mongodb, mongoose, baucis]
image:
  feature: /images/diary-bg20.jpg
share: true
---

In previous post, we've seen how to navigate in our site using
backbone.js. In this post, we will go back to server side, and build
our blog api using [baucis](https://github.com/wprl/baucis). At the
end of this tutorial, you will be able to query your mongodb database
and pull blog information.

# Building Schemas with Mongoose

Firstly, let's define our blog schema. We will use
[mongoose](http://mongoosejs.com) to interact with mongodb. Go to
app/scripts directory and create a new file named schemas.js.

*app/scripts/schemas.js*

{% highlight javascript %}
var mongoose = require('mongoose');
var baucis = require('baucis');

// Define journal schema
var journalSchema = mongoose.Schema({
    paper: {name: String, url: String},
    author: {name: String, photo: String },
    title: String,
    body: String,
    date: { type: Date, default: Date.now},
    meta: {
        votes: Number,
        favs: Number,
        tags: { type: String, lowercase: true, trim: true }
    }
});

// db connection string
var MONGO_URI = process.env.MONGOHQ_URI || 'mongodb://localhost/test';


// connect to mongodb with mongoose
mongoose.connect(MONGO_URI);

// define a journal model with journalSchema
mongoose.model('journal', journalSchema);

// use baucis to create a rest api
var controller = baucis.rest('journal');

{% endhighlight %}

We created a blog schema, connected to our database, defined a model
based on our schema, and finally used baucis to create the rest api.
Now if you run your server, your api will be available at:

*http://localhost/api* endpoint.

Here is a table, describing available functions:

| HTTP Verb     | /journals   | /journals/:id |
| ------------- | ------------- | --------------- |
| GET           | Get all or a subset of documents | Get the addressed document |
| POST          | Creates new documents and sends them back.  You may post a single document or an array of documents.      | n/a |
| PUT           | n/a | Update the addressed document |
| DELETE        | Delete all or a subset of documents | Delete the addressed object |

To test it go to *http://localhost/api/journals* in your browser, it
should return an empty array. Because we don't have any blogs in our
database yet.

# Using Mongo Shell

Though we can post blogs to our database using the api, we will simply
create blogs using the mongo shell. Mongo shell is CLI for using
mongodb. run *mongo* in your console to open the shell. Issue the 
following commands to create your first blog entry.

{% highlight bash %}
   # This is for commenting
   # just ignore it.

   # List databases
   show dbs
   
   # Choose database, test
   use test

   # List collections in the database
   show collections

   # journals collection is created when we created a model with mongoose.
   # collection is the plural form of our model.

   #list the contents in journals collection
   # it should be empty
   db.journals.find()

   # place a new blog entry in journals collection
   # here we define only title and body attributes, you can define other attributes as well

   db.journals.create({title: 'My Simple Blog 1', body: 'Simple body for testing' })

   # list the contents again to see the newly created blog
   db.journals.find()
   
   
{% endhighlight %}

Mongo shell is interpreting javascript code here, so use
javascript. Undefined attributes will have their defaults value We can
disable null values for attributes, but for simplicity we don't.

Now go to http://localhost/api/journals again and see the newly created blog entry.

> In the next post, we will consume this api with backbone.js and list
the blogs in our main page.
