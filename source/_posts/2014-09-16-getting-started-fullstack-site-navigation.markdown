---
layout: post
title: "Getting Started Fullstack: Site Navigation"
date:   2014-01-01 09:52:06
comments: true
categories: nodejs backbonejs
tags: [nodejs, backbonejs]
image:
  feature: /images/diary-bg20.jpg
  background: colorflags-bg.jpg
---

In previous post we've seen how to setup a node.js server and serve our index.html.
In this post, we will implement navigation in our site using backbone.js. At the end of this tutorial,
you will be able to navigate between tabs in your site.

# Router

Backbone.Router provides methods for routing client-side pages. We will start by creating a custom router
class. Place this in *js/app.js*

{% highlight javascript %}
  var AppRouter = new (Backbone.Router.extend({
    initialize: function() {

    },
    
    routes: {
        
    }
}))();
{% endhighlight %}

When creating a router you may pass its routes hash directly, or manually create a route for the router.
We will do it manually inside initialize function. But first let's define our routes in an object.

*js/app.js*
{% highlight javascript %}
   var NavigationRoutes = {
    "Home": {
        route: "",
        view: Home
    },
    "About": {
        route: "About",
        view: About
    },
    "default": {
        route: "*default",
        view: NotFound,
        hide: true
    }
}
{% endhighlight %}

NavigationRoutes object will keep our routes for the application. The keys are title for the route. "route" property
is the routing string. "view" property defines the backbone view objects that will be rendered for the route, they are the *Route Views* We will
define them later. Finally "default" key has a "hide" property telling that it will be hidden on navigation tab.
We can add different properties as we like here, for example "permissions" property might tell a navigation tab 
to be only shown for an admin.

Next we will add these routes to our router. Place this function in your Backbone router.

*js/app.js*
{% highlight javascript %}
   initialize: function() {

        this.route(NavigationRoutes['default'].route, 'default');

        for (var key in NavigationRoutes) {
            if (key != "default")
                this.route(NavigationRoutes[key].route, key);
        }
    },
{% endhighlight %}

Here we traverse NavigationRoutes, and add each route in our router. There is a little logic there for the default 
route. We add the default route first, so it becomes the last route to be looked.
The default route is defines as "*default", this is called a splat and matches everything in the url. So anything else
that is not defined previously will be matched by default route, and NotFound view will be rendered.

# Navigation View

## Bootstrap NavBar

Before we dive into Backbone.View, let's see our html code for the navigation bar. Place this inside your index.html


*index.html*
{% highlight html %}
    <nav class="navbar navbar-default">
      <div class="navbar-header">
        <button class="navbar-toggle" type="button" data-toggle="collapse" data-target="#navbar-collapse">
          <span class="sr-only"> Toggle navigation</span>
          <span class="icon-bar"></span>
          <span class="icon-bar"></span>
          <span class="icon-bar"></span>
        </button>
        <a class="navbar-brand" href='#'> Simple Blog </a>
      </div>

      <div class="collapse navbar-collapse" id="navbar-collapse">
           
        <ul class="nav navbar-nav" id="nav-item-container">
        <!-- Navigation tabs will be inside #nav-item-container here -->
        </ul>
      </div>

    </nav>

     <div class="container" id="container">

    </div>
{% endhighlight %}

Our Backbone.View for the navigation tabs will control the *#nav-item-container*.
*#container* element will contain the navigated content.

## Navigation Bar View

   Backbone views don't determine anything about HTML, it can be used with any templating library. Underscore.js in this
case. Place this in *js/views/nav.js*

*js/views/nav.js*
{% highlight javascript %}
var Navbar = Backbone.View.extend({
    initialize: function(options) {

                // we pass the NavigationRoutes as an option, later we traverse the routes in render function.
        this.routes = options.routes;


        // router will fire route event whenever url changes, we rerender our view
        // on each url change
        Backbone.history.on('route', function(source, path) {
            this.render(path);
        }, this);
    },

    // events hash that handles events in our view
    events: {
                // click handler for a navigation tab
        'click a': function(source) {
            var hrefRslt = source.target.getAttribute('href');
            Backbone.history.navigate(hrefRslt, {trigger: true});

            // Cancel the regular event handing so that we won't actually
            // change URLs We are letting Backbone handle routing
            return false;
        }
    },

    render: function(route) {
            // Clear the view element
        this.$el.empty();

        // this is the underscore template for a navigation tab.
        var template = _.template("<li class='<%=active%>'><a href='<%= url%>'><%=visible%></a></li>");

        // Traverse the NavigationRoutes
        for (var key in this.routes)
        {
                // don't render if route is hidden
            if (!this.routes[key].hide)
                this.$el.append(template({url: this.routes[key].route, visible: key, active: route === key ? 'active': ''}));
        }
    }
});
{% endhighlight %}

   Backbone.View's looks complicated at first, render function is used to render the actual view, in our case, its about manipulating the dom. The dom is referenced by *this.el*, equal to 
*#nav-item-container*. *this.$el* references the jquery dom object, equal to *$('#nav-item-container')*. We don't set *this.el* in this code
we will pass it as parameter when we initialize our view later.

   *template* variable defines an [underscore template](http://underscorejs.org/#template). It is our navigation tab. A json model is passed to the template to render actual html.
   Finally html is appended to *this.$el*. This is done for each element in *NavigationRoutes*. We can add any logic we wan't here to render, for now,
we only use *hide* property for the *NavigationRoutes* to control if a link should be rendered or not.


## Navigated Content View

Next, we define a view for navigated content. It will control the *#container* element.

*js/views/nav.js*
{% highlight javascript %}
var Content = Backbone.View.extend({
    initialize: function(options) {
                // pass the NavigationRoutes as an option
        this.routes = options.routes;

        // Listen for route changes, and render when a route changes.
        Backbone.history.on('route', function(source,path) {
            this.render(path);
        }, this);
    },

    render: function(route) {
        this.$el.html((new this.routes[route].view()).render().el);
    }
});
{% endhighlight %}

This is a bit simpler, tricky part is the render function. Here, we initialize the *view* property defined in our *NavigationRoutes*, *view* contains
the Backbone.View elements for each route. Call the render method for the View element. *render* method will return the Backbone.View, (we will define that later) , so we can access the el property. Finally we set *this.$el.html* to the dom element of the route view.

Phew, that's quite hard to explain, hope it becomes clear later.

## Individual Route Views

Next, we define the route views, these will be rendered into the *#container*. Place this in your *js/views/main.js*.

*js/views/main.js*
{% highlight javascript %}
var Home = Backbone.View.extend({


    template: _.template($('#home-template').html()),
    
    initialize: function () {
        
    },

    render: function () {
            // Set the html to the template
        this.$el.html(this.template());

        // We return this, so parent view can access the this.el property
        return this;
    }
});


var About = Backbone.View.extend({

    template: _.template($('#about-template').html()),
    
    initialize: function () {
        
    },

    render: function () {
        this.$el.html(this.template());
        return this;
    }
});


var NotFound = Backbone.View.extend({

    template: _.template($('#404-template').html()),
    
    initialize: function () {
        
    },

    render: function () {
        this.$el.html(this.template());
        return this;    }
});
{% endhighlight %}

Each view defines an underscore template property. Templates are initialized with some html from jquery dom elements. We will define them later in our index.html. *render* method simply sets the view's el property to the underscore template, and returns the view which was used in our navigated content view.

### Route View Templates

We define these templates in our index.html.

*index.html*
{% highlight html %}

    <script type='text/template' id='home-template'>
       Home
    </script>

    <script type='text/template' id='about-template'>
      About
    </script>

    <script type='text/template' id='404-template'>
      Oops..
    </script>

{% endhighlight %}

 Since these templates should not be readily visible in the page, we wrap them in script tags and give each an id. We referenced these templates
with jquery before.

While we're at it, let's add the new script files we've created to the index.html as well.

*index.html*
{% highlight html %}
      <script src="/js/views/nav.js"></script>
      <script src="/js/views/main.js"></script>
      <script src="/js/app.js"></script>
{% endhighlight %}

# Main Function

*js/app.js*
{% highlight javascript %}
   $(document).ready(function () {

    new Navbar({el: $('#nav-item-container'), routes: NavigationRoutes});
    new Content({el:$('#container'), routes: NavigationRoutes});


    Backbone.history.start({pushState: true});
});
{% endhighlight %}

Finally the main function, initializes *Navbar* and *Content* views. And calls *Backbone.history.start*
We pass the el property for the views here as mentioned previously. and *NavigationRoutes* is passed as an option.

By default backbone routes the url's with a containing # symbol. 
*Backbone.history.start* is passed *pushState:true* parameter, to get rid of the #.

> Congratz, now you have a working implementation of a navigated content using backbone.js

# Final Touch

As a final touch, let's put some html inside our *#home-template* so it looks more like a blog.

*index.html*
{% highlight html %}
   
    <script type='text/template' id='home-template'>

      <div class="row">
        <div class="col-lg-12">
          <h1 class="page-header"> Blog Home <small> Blog Homepage</small></h1>
          <ol class="breadcrumb">
            <li><a href="/">Home</a></li>
            <li class="active">Blog Home </li>
          </ol>
        </div>
        
      </div>

      <div class="row">
        <div class="col-lg-8" id="blog-list-view">

          
          <ul class="pager">
            <li class="previous"><a href="#">&larr; Older</a></li>
            <li class="next"><a href="#">Newer &rarr;</a></li>
          </ul>
          
          <div id="blog-list-container"></div>
          
          <ul class="pager">
            <li class="previous"><a href="#">&larr; Older</a></li>
            <li class="next"><a href="#">Newer &rarr;</a></li>
          </ul>
        </div>

        <div class="col-lg-4">
          <div class="well">
            <h4> Blog Search </h4>
            <div class="input-group">
              <input type="text" class="form-control">
              <span class="input-group-btn">
                <button class="btn btn-default" type="button">&nbsp;<i class="fa fa-search"></i> </button>
              </span>
            </div>
          </div>

          <div class="well">
            <h4>Popular Blog Categories</h4>
            <div class="row">
              <div class="col-lg-6">
                <ul class="list-unstyled">
                  <li><a href="somestuff">some stuff</a></li>
                  <li><a href="somestuff">some stuff</a></li>
                  <li><a href="somestuff">some stuff</a></li>
                </ul>
              </div>
            </div>
          </div>
        </div>
      </div>
      
      
    </script>

{% endhighlight %}