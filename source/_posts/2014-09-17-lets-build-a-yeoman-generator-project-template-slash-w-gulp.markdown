---
layout: post
title: "Let's Build a Yeoman Generator: Project Template /w gulp"
date: 2014-09-17 12:14:32 +0300
comments: true
categories: nodejs gulp yeoman phaser
tags: [nodejs, gulp, yeoman, phaser]
comments: true
share: true
image:
  feature: /images/diary-bg20.jpg
---

## Introduction

[Yeoman](http://yeoman.io) is a project scaffolding tool, it defines your work-flow, and scaffolds the boilerplate. [Gulp](http://gulpjs.com) is a build system, alternative to grunt, it is stream based, and prefers code over configuration.

In this two series articles we will learn how to build a yeoman generator from scratch, using test driven development. for Phaser game projects, which I before introduced in my previous article. We will use gulp as our build tool and requirejs for module loader.

First step towards a yeoman generator, is building a project template. yeoman is only responsible for scaffolding. The rest is building the project with gulp and having a project template. In this article we will build our project template using gulp and requirejs, in the next article we will introduce yeoman, and start test driven development.

## Initialize Phaser Game Project

The Phaser is a game framework that runs on browser, as we develop our game, we will run the game on the browser and live reload the changes. We will use npm for our development dependencies, and bower for the front end dependencies.

First we initialize npm.

{% highlight bash %}
  $ npm init
{% endhighlight %}

This will ask various questions and create a `package.json` file for us.

Next we initialize bower the same way, and create a `bower.json` file.
{% highlight bash %}
  bower init
{% endhighlight %}

## Decide a directory structure

The next step is to decide a directory structure for the project.

{% highlight bash %}
.
|-- app
|   |-- data
|   |-- index.html
|   `-- scripts
|       |-- app.js
|       |-- common.js
|       |-- config.js
|       |-- lib_main.js
|       |-- main.js
|       |-- prefabs
|       `-- states
|           |-- boot.js
|           `-- preload.js
|-- bower.json
|-- build
|   |-- wrap.end
|   `-- wrap.start
|-- gulpfile.js
`-- package.json

{% endhighlight %}

`app` folder is where our application code resides. `build` folder is for files used by requirejs optimizer. We have our `package.json`, `bower.json` and `gulpfile.js`. The project will compile into `public` directory as we will see later.

### Configure bower and jshint

npm will install it's dependencies in the `node_modules` folder, and bower will install it's dependencies in the `bower_components` folder, but, yet we can configure the default directory for bower using a `.bowerrc`

Create a `.bowerrc` file and add these in.

`File: .bowerrc`

{% highlight bash %}
{
  "directory": "app/bower_components"
}
{% endhighlight %}

Next, create a `.jshintrc` file. and setup your configuration.

Finally install the bower dependencies and save them in the `bower.json`.

{% highlight bash %}
  $ bower install requirejs almond phaser --save
{% endhighlight %}

Note we use `--save` flag to make our changes saved in `bower.json`. We have three dependencies, requirejs the module loader, almond that we discussed in the previous article about requirejs, and Phaser the game framework.

## gulp.js, The streaming build system

We have three types of tasks that gulp can help us for our project. Building the project, linting, and injecting code to our project, that is injecting our bower dependencies to our requirejs configuration file, and serving the project and watching for changes.

We will use gulp plugins and other npm modules to help us, the first plugin we use is `gulp-load-plugins`, that automatically loads any gulp plugins we have in our `package.json`, that is convenient.

{% highlight js %}
var gulp = require('gulp');
var $ = require('gulp-load-plugins')();
var rimraf = require('rimraf');
var runSequence = require('run-sequence');
{% endhighlight %}

Note that we assign `gulp-load-plugins` to `$`, that is all the plugins are attached to. We also require two other npm modules, `rimraf` for removing files/folders, and `runSequence` for running gulp tasks in sync. Because gulp runs tasks asynchronously by default.

### gulp; build the project!

First task is simple, `clean`, that will remove the build folder.

{% highlight js %}
gulp.task('clean', function(cb) {
  rimraf(paths.dist, cb);
});
{% endhighlight %}

This is how you define a gulp task. notice we pass `cb` as a parameter, that makes the gulp task asynchronous. It will be called once the task is done and tell gulp to continue with the other tasks. By default gulp will run all the tasks and wait for nothing.

Next task is building the extra files in the `app` directory. In our case it's `index.html`, you might have extra `favicon.ico` or `robots.txt`.

{% highlight js %}
gulp.task('build-extras', function() {
  gulp.src('app/*.*')
      .pipe(gulp.dest('public'));
});
{% endhighlight %}

`gulp.src` takes a glob and represents a file structure, which then can be piped to plugins. `gulp.dest` can be piped to and it will write files. In our case we take all the files in app folder, `app/*.*`, and write them into `public` folder.

Next task is building the scripts. In our case this is moving the scripts into the `public` directory. We might have used minification, optimization and concatenation, we don't need it. We have an extra task that uses requirejs optimizer to optimize the script files and build a stand-alone code you can check out the [source code](https://github.com/eguneys/phaser-gulp-requirejs) for this article.

{% highlight js %}
gulp.task('build-commonjs', function() {
  var mergeStream = require('merge-stream');

  var bowers = gulp
      .src(['app/bower_components/**/*.js'])
      .pipe($.changed('public/scripts/lib'))
      .pipe(gulp.dest('public/scripts/lib'));

  var common = gulp.src(['app/scripts/**/*.js'])
      .pipe($.changed('public/scripts/app'))
      .pipe(gulp.dest('public/scripts/app'));

  return mergeStream([bowers, common]);
});
{% endhighlight %}

Note that, we use `merge-stream` that combines two gulp streams together, and we return the stream. Remember we passed an extra parameter, `cb` to gulp that made the task asynchronous, this is a second way of doing that, returning the stream also makes the task run asynchronous. The third way of making a task asynchronous is to return a promise. For more information about gulp API see [docs](https://github.com/gulpjs/gulp/blob/master/docs/API.md).

Finally we have our `build` task, that will run the `clean`, `jshint-fail`, and `build-requirejs` tasks synchronously in that order.

{% highlight js %}
gulp.task('build', function(cb) {
  runSequence('clean', 'jshint-fail', 'build-commonjs', cb);
});

gulp.task('default', ['build']);
{% endhighlight %}

Note that, we also have a special `default` task, that will run with only `gulp` command, opposed to `gulp build`.

By default, tasks are run concurrently, as opposed to grunt which runs the tasks in series. To run a list of tasks in sync either tasks have to be asynchronous and have to depend on each other or you need to run tasks via `runSequence`.

### gulp; lint, and inject some code!

Next task, is linting our code with `jshint`, there is a gulp plugin we access it using `$.jshint`, remember `gulp-load-plugins`.

{% highlight js %}
gulp.task('jshint-fail', function() {
  return gulp.src(['gulpfile.js',
    'app/**/*.js',
    '!app/bower_components/**'])
        .pipe($.jshint())
        .pipe($.jshint.reporter('jshint-stylish'))
        .pipe($.jshint.reporter('fail'));
  });
{% endhighlight %}

Note that we pipe the jshint plugin and the jshint reporters to the gulp stream. Visit [here](https://github.com/yeoman/bower-requirejs) for more information on jshint plugin.

There is an npm module [bower-requirejs](https://github.com/yeoman/bower-requirejs), that wires up installed bower components into requirejs config, we also use it in our gulp task, though I won't show it here.

### gulp; serve and watch files!

Next task, is serving our project to the browser, we use [gulp-webserver](https://github.com/schickling/gulp-webserver) plugin for that. It will start a connect server and serve our files as well as watch them and run a live reload server and inject a live reload script so the browser will auto refresh when a file is changed.

Final task is `watch` task, that will watch our source files for changes and run various tasks on change.

{% highlight js %}
gulp.task('watch', ['serve'], function() {
  gulp.watch('app/scripts/**/*.js', ['build-commonjs']);

  gulp.watch('app/*.*', ['build-extras']);
    
  gulp.watch('bower.json', ['bower-rjs']);

  gulp.watch('gulpfile.js', function() {
    console.log($.util.colors.red('\n----------' +
                                  '\nRestart the Gulp process' +
                                  '\n----------'));
  });
});
{% endhighlight %}

You might wonder, `gulp-webserver` is also watching the folders, why we need gulp to watch again. Well gulp watches the source files, and runs tasks that compiles the source into the `public` folder, meanwhile `gulp-webserver` watches the `public` folder for changes, and refreshes the browser on change.

Also note that 'watch' task depends on 'serve' task, we tell that by passing a dependency array as the second argument to `gulp.task`.

Here are the list of dependencies we've used in this article and the command to install and save them in the `package.json`.

{% highlight bash %}
 $ npm install --save-dev gulp gulp-load-plugins gulp-changed gulp-imagemin gulp-livereload gulp-webserver gulp-util gulp-jshint jshint-stylish rimraf merge-stream run-sequence bower-requirejs requirejs
{% endhighlight %}

There are many gulp plug-ins listed in the [official website](http;//gulpjs.com/plugins). Yet gulp is sensitive about plugins, that there should be one, if it is only necessary, and they have a [blacklist of plugins](https://github.com/gulpjs/plugins/blob/master/src/blackList.json) that should not be used, so pick your plugins carefully, and use bare npm modules wherever possible. Also checkout the [documentation](https://github.com/gulpjs/gulp/blob/master/docs/README.md) and [recipes](https://github.com/gulpjs/gulp/tree/master/docs/recipes) for achieving various tasks.

## Conclusion

In this article we have built a project template for Phaser using gulp as our build tool, and requirejs as our module loader. We have learned how to define gulp tasks and how to make use of gulp plug-ins to enable us a nice development environment, with watching files and live-reloading our changes. Note that yet we used Phaser as a sample project, this will give you the main idea, and hopefully you will apply the same principles to your specific projects. You can find the full source code at [github](https://github.com/eguneys/phaser-gulp-requirejs).

In the next article we will use this project template to build a yeoman generator, we will also show test driven development using mocha and yeoman test helpers.
