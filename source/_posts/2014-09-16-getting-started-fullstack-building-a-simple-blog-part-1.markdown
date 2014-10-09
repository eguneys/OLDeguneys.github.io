---
layout: post
title: "Getting Started Fullstack: Setting Up The Server"
date: 2013-12-31 00:52:07
categories: nodejs npm bower mongodb
tags: [nodejs, npm bower mongodb]
comments: true
image:
  feature: /images/diary-bg20.jpg
  background: colorflags-bg.jpg
---

Getting Started Full Stack - Building a simple blog - Part 1


 In this post, we will install the required components to build a simple blog. In the end of this post, you will be serving a hello world page, with a responsive bootstrap theme, and all necessary script files.

# Server Side:

[Node.js](http://nodejs.org)
: Host the website, on javascript platform, based on Google's [V8](https://code.google.com/p/v8/) JavaScript Engine

[Express](http://expressjs.com/guide.html)
: Web application framework for node.js


[Mongodb](http://docs.mongodb.org/manual/installation)
: Open-Source NoSQL database written in C++. Also look at [Mongoosejs](mongoosejs.com/docs/index.html), simple mongodb driver for nodejs.

[Bower](http://bower.io)
: Package manager for the web. Simplifies installing client side resources.


# Client Side:
[Bootstrap](http://getbootstrap.com/getting-started)
: Responsive front-end css framework.

[JQuery](http://jquery.com/download)
: Javascript, for interactive webpages.

[Backbone.js](http://backbonejs.org)
: Gives structure to web applications. Models, Collections and Views.

# Let's get Started

## Introduction

Building a simple blog, solving problems along the path, it's gonna be fun.

### Web Server
Nodejs hosts the website. Serving pages, scripts and stylesheets.
Expressjs is a web application framework, simplifies dealing with serving pages.
It uses middleware to extend functionality like plugins.

### Database
Mongodb our nosql database. it has no relational tables, but nested documents.
Mongoosejs is our driver for mongodb on nodejs. Model schemas and operate.

### Content Management
Bower downloads scripts and stylesheets from web into our path, we will simply reference it.

### CSS Theme
Bootstrap gives the site a nice look. Responsive, meaning it will behave when resized, looks nice on any device.

### Javascript
JQuery, main dependency for manipulating dom.
Backbone.js is mvc framework for building client side interaction.

> Backbone.js is lightweight, and heavily depends on Jquery, a popular alternative is [angularjs](http://angularjs.org). Angular.js doesn't need Jquery, and is a higher level framework.

> Backbone.js doesn't implement templating. Templating is supported by mainly [underscore.js](http://underscorejs.org/#template) or other libraries. ([handlebarsjs](http://handlebarsjs.com/), [mustache](http://mustache.github.io/))

> Backbone.js will teach us more about structure, templating, jquery, and javascript. We should move to higher level frameworks later on. (possibly built on backbonejs) Finally, backbonejs isn't bound to SPA's, we may use it for other projects later on (more coming after this).

## Installation

`I use ubuntu/linux you should find a way to install on your platform, read the guides and google.`

## Npm, Node.js

{% highlight bash %}
# install node.js
sudo add-apt-repository -y ppa:chris-lea/node.js
sudo apt-get update
sudo apt-get install nodejs

# test should look like: v0.10.12
node -v
{% endhighlight %}

Or,
follow the [Instructions](https://github.com/joyent/node/wiki/Installing-Node.js-via-package-manager)

This will also install npm package manager. Npm is used to install packages required by node.js.

## Mongodb

{% highlight bash %}
# install mongodb
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 7F0CEB10
echo 'deb http://downloads-distro.mongodb.org/repo/ubuntu-upstart dist 10gen' | sudo tee /etc/apt/sources.list.d/mongodb.list
sudo apt-get update
sudo apt-get install mongodb-10gen

# test should bring mongo shell, Ctrl-c to break
mongo
{% endhighlight %}

More [Instructions](http://docs.mongodb.org/manual/installation/)

## Bower

{% highlight bash %}
#install bower
npm install -g bower
# test should look like: 1.2.8
bower -v 
{% endhighlight %}

More [Instructions](http://bower.io)

>Congratz, this is enough to host our simple blog. We will learn these tools along the way. Now let's build our codebase.

# Directory Structure

Build the following directory structure. public folder will host our public pages, scripts and stylesheets. 'scripts' folder will host javascript code for running the nodejs server, interacting with mongodb. 

{% highlight bash %}
../mysimpleblog
`-- app
    |-- public
    |   `-- js
    |       |-- collections
    |       |-- models
    |       |-- routers
    |       `-- views
    `-- scripts
{% endhighlight %}

# Configuration

## Package.json

{% gist eguneys/e0ae625587d0f578a8e0 %}

[package.json](http://package.json.nodejitsu.com/) is a packaging format for node.js.
Important part here is dependencies, where we declare dependencies for our project.
Place it in root folder of your project, where your app folder resides.

[express](expressjs.com) : web application framework, It uses middleware to handle http requests.
Middleware could be third party code, which we will use shortly.

[baucis](https://github.com/wprl/baucis) : Baucis is Express middleware that creates configurable REST APIs using Mongoose schemata

[connect](https://github.com/senchalabs/connect) : Lower level middleware for node.js, not important for us any time soon. baucis needs it so we include.

## .bowerrc

{% gist eguneys/61b1ed8e2fee1d347c2f %}

[.bowerrc](http://stackoverflow.com/questions/14079833/how-to-change-bowers-default-components-folder) file will change bower's default components folder. Place it in root folder of your project.

# Installing packages

Server side dependencies are defined in package.json, and installed with npm. Client side dependencies are defined in bower.json and installed with bower.

However we won't be using bower.json rather install the required files manually.


{% highlight bash %}
# install server side packages (defined in package.json)
npm install
#install client side packages manually with bower
bower install bootstrap
bower install fontawesome

bower install jquery
bower install backbonejs
bower install underscore

{% endhighlight %}

Server side packages can be directly used in javascript code, we will see how that's done later. Client side packages are simply files downloaded in directory we defined in .bowerrc so you can check it out there. We will reference them in our index.html.

# Setting up the server

Final step is to actually setup the server, so we can host our files. Go to app/scripts directory and create the two files below.

{% gist eguneys/938cd7857d3480f63d7a %}

Further information:
[middleware](http://stackoverflow.com/questions/12695591/node-js-express-js-how-does-app-router-work) , 
[require](http://stackoverflow.com/questions/5311334/what-is-the-purpose-of-nodejs-module-exports-and-how-do-you-use-it) .

Finally we need an index.html to serve, and we are ready to launch. Go to app/public directory and create the file below.

{% gist eguneys/917ed8e8f24570f696fa index.html %}

# Launch Time
To run the server go to app folder and run:
{% highlight bash %}
   # test: output should be 
   # - listening on port 3000
   node ./scripts/web
{% endhighlight %}

Now launch your browser and locate to http://localhost:3000.

One final note here is that, if you make changes inside app/public
folder, a browser refresh is enough to see the changes, if you make
changes inside app/scripts folder, you need to restart the server, in
order to changes to take effect. There are tools to live reload every
change but that's for a later time.

> Please share your thoughts below. Thank you.
