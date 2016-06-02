---
layout: post
title: "Gulp Workflow For Building Web Projects"
date: 2016-06-02 17:51:35 +0300
comments: true
categories: nodejs, npm, gulp
tags: [nodejs, npm, gulp]
image:
  feature: /images/diary-bg20.jpg
  background: colorflags-bg.jpg
share: true
---

Gulp Workflow For Building Web Projects

In this quick tip, we will build a simple and fast gulp workflow for building web projects.

If you are building a small web project, or testing an idea, and you just want to quickly get up and running, you will find gulp useful to help you handle your workflow tasks.

You can also check out the example [github repo](https://github.com/eguneys/gulpp).

## Gulp Tasks

This is a list of essential gulp tasks we will setup:

Development
* Linting
* Bundle ES6/JavaScript with source maps
* Watch JavaScript and bundle on changes

Production
* Bundle and minify ES6/JavaScript
* Minify CSS
* Copy files

For development, we need to bundle ES6 code for the browser, and include source maps, so we can easily debug it in the browser. We also watch source files for changes and re-bundle so we can quickly see the result of our changes. Optionally, we lint our code, so it helps detect errors in our code, we add it because it's simple.

For production, again we bundle ES6 code for the browser, this time, we minify it. We also minify our CSS. Finally, we copy other assets to our build folder.

This is the simplest setup we need to get things working. Use `http-server` to serve our project for development, and reload browser manually when the code changes, so we don't have to complicate our setup to handle these jobs.

```bash
npm install http-server -g
```

## Setting Up The Project

Setup the file structure, and initialize our Node.js project, it will add a `package.json` to our project.

```bash
mkdir examples assets src
npm init
```

### Development

Install [npm](https://www.sitepoint.com/beginners-guide-node-package-manager/) dependencies for development:

```bash
npm install --save-dev gulp gulp-util gulp-streamify gulp-jshint jshint gulp-uglify vinyl-source-stream watchify babelify babel-preset-es2015 babel-plugin-add-module-exports gulp-csso
```

We have `gulp` and some gulp plugins to handle gulp tasks. `babelify` is [browserify transform](https://github.com/substack/node-browserify#browserifytransform) to transform ES6 code with [babel](https://babeljs.io/) and bundle it for the browser. `babel` needs some plugins to work properly, so we include them as well. `watchify` is [browserify plugin](https://github.com/substack/node-browserify#plugins) for watching source files, and re-bundle on code changes. and `vinyl-source-stream` is for gluing `browserify` with `gulp`.


Now add an `examples/index.html` file to test our project:

```html
<html lang="en">
  <head>
    <title>Gulp workflow for setting up a web project</title>
    <link rel="stylesheet" href="../build/main.css">
    <script src="../build/gulpp.js"></script>
  </head>

  <body>
    <div id="gulpp"></div>
    <script>
      (function() {
        var ground = Gulpp(document.getElementById('gulpp'));
      })();
    </script>
  </body>
</html>
```

In the head section, we reference a CSS file and a JavaScript file from the `build` folder. This folder and the files will be created by our build workflow. In the body section, we have a single div element `#gulpp`. Finally inside the `script` block we have a [self invoking function](https://www.sitepoint.com/community/t/understanding-anonymous-functions/41542/10). Our code is exposed with the `Gulpp` global name, we will see later how this is done. We can use it inside this function to test our code, we call it as a function and pass the `#gulpp` DOM element, so that our app will hopefully render into it.

Run `http-server` to serve our project folder, and visit `http://localhost:8080/examples/index.html`.

There are three errors, we need to build CSS and JavaScript and put them in the `build` folder:

```
Cannot GET http://localhost:8080/build/main.css
Cannot GET http://localhost:8080/build/gulpp.js
Uncaught ReferenceError: Gulpp is not defined
```

Add a `gulpfile.js` file in the root of our project to define gulp tasks:

```js
var source = require('vinyl-source-stream');
var gulp = require('gulp');
var gutil = require('gulp-util');
var jshint = require('gulp-jshint');
var watchify = require('watchify');
var browserify = require('browserify');
var uglify = require('gulp-uglify');
var streamify = require('gulp-streamify');

var sources = ['./src/main.js'];
var destination = './build';
var onError = function(error) {
  gutil.log(gutil.colors.red(error.message));
};

var standalone = 'Gulpp';

gulp.task('dev', function() {
  var opts = watchify.args;
  opts.debug = true;
  opts.standalone = standalone;
  var bundleStream = watchify(browserify(sources, opts))
      .transform('babelify',
                 { presets: ["es2015"],
                   plugins: ['add-module-exports'] })
      .on('update', rebundle)
      .on('log', gutil.log);

  function rebundle() {
    return bundleStream.bundle()
      .on('error', onError)
      .pipe(source('gulpp.js'))
      .pipe(gulp.dest(destination));
  }

  return rebundle();
});

gulp.task('default', ['dev']);
```

First, we require our dependencies, next we define where our source files are and our destination folder, where our bundled project is put.


Then we define our `dev` task. `browserify` will take our source files, and bundle them into a single file. `debug` option is for generating source maps and `standalone` option will export the code as a window global. `babelify` will transform ES6 before bundling. `watchify` will emit `update` event when the files change, and `log` event when the bundle is created.

We define a `rebundle` function to do the actual bundling and call it on `update` event so it will re-bundle on source changes. After calling `bundle` on the browserify instance, it will produce a single JavaScript file. Next, we add `vinyl-source-stream` passing a filename `gulpp.js` in to the mix to [make it more gulp friendly](http://stackoverflow.com/a/30851219/3995789), and finally write it inside `./build` folder. See [this article](https://www.sitepoint.com/getting-started-browserify/) for a more detailed explanation of how this works. Also notice that we added handlers for the `log` and `error` events, so the bundle errors and logs can be logged.

We define `dev` task as the `default` task for gulp. So run `gulp` to test if our gulp task is working:

```bash
[11:41:37] Cannot find module '/projects/gulpp/src/main.js' from '/projects/gulpp'
```

It's working, but we need to add a JavaScript source file to bundle.

Add a `src/main.js` file to define our application starting point:

```js
import view from './view';

const ctrl = {
  data: {
    face: 'cats',
    msg: '[gulpp]'
  }
};

function render(element, obj) {
  var tag = obj.tag;
  var body = obj.children;
  element.innerHTML = `<${tag}>${body}</${tag}>`;
}

function init(element, config = {}) {
  var msg = config.msg || "[gulpp]";

  ctrl.data.msg = msg;

  ctrl.data.render = () => {
    render(element, view(ctrl));
  };
  ctrl.data.renderRAF = function() {
    window.requestAnimationFrame(ctrl.data.render);
  };
  ctrl.data.render();
}

module.exports = init;
```

`src/main.js` imports `./view` module, So add a `src/view.js` file to define it.

```js
module.exports = function(ctrl) {
  var data = ctrl.data;
  return {
    tag: 'p',
    children: `${data.msg} is alive! <br/> <span class="cat">${data.face}</span> `
  };
};

```

To simply describe our code, we set up an MVC style project. `./view` module takes the `ctrl.data` and returns a data structure so we can render it. `./main` module sets up the `ctrl.data` and gives it to `render` function, so it renders it to DOM. Remember the `Gulpp` global variable inside our HTML file, we called is as a function and passed a DOM element. It is what we export from inside `src/main.js`, in this case the `init` function.


If we want to add an external dependency, first we install it:

```bash
npm install --save cat-ascii-faces
```

Then import and use it (inside `src/main.js`):

```js
import cats from 'cat-ascii-faces';

setInterval(() => {
  ctrl.data.face = cats();
  ctrl.data.renderRAF();
}, 1000 + (Math.random() * 4000));
```

This will update our `ctrl.data` and re-render our app.

Run gulp `dev` task if it's not already running, and refresh the browser to see the code working. We can see the ES6 source files and set breakpoints on browser:

{% img https://dab1nmslvvntp.cloudfront.net/wp-content/uploads/2016/05/1464275958gulp-workflow-sourcemap.png [gulp workflow source-map breakpoint] %}

If we want to serve assets like stylesheets or images, we can copy these files into build folder using a gulp task:

```js
gulp.task('assets', function() {
  var assetFiles = './assets/**/*';
  var path = require('path');
  gulp.src(assetFiles)
    .pipe(gulp.dest(path.join(destination)));
});
```

We can hook this task to `dev` task so it will run every time we run `dev` task. Change our `dev` task:

```bash
-gulp.task('dev', function() {
+gulp.task('dev', ['assets'], function() {
```

Finally, add a gulp task to lint our JavaScript:

```js
gulp.task('lint', function() {
  return gulp.src(sources)
    .pipe(jshint({ esversion: 6 }))
    .pipe(jshint.reporter('default'));
});
```

Note the `esversion: 6` option we pass to `jshint` so it works with ES6. All setup for development. Write some code and reload the browser to see the changes.

### Production

We need to minify code for production use.

First, add a task to minify CSS:

```js
gulp.task('css', function() {
  var cssFiles = 'assets/*.css';

  return gulp.src(cssFiles)
    .pipe(csso())
    .pipe(gulp.dest(destination));
});
```

Add some CSS files in `assets` folder, then run `gulp css` to run this task, and see the minified output in `build/main.css`.

Next, add a task to minify JavaScript:

```js
gulp.task('prod', ['css'], function() {
  return browserify(sources, {
    standalone: standalone
  }).transform('babelify',
               { presets: ["es2015"],
                 plugins: ['add-module-exports'] })
    .bundle()
    .on('error', onError)
    .pipe(source('gulpp.min.js'))
    .pipe(streamify(uglify()))
    .pipe(gulp.dest(destination));
});
```

This task is similar to our `dev` task, except we don't have the `debug` option, and we have an extra step for minifying code, using `gulp-uglify` and we have to wrap that with `gulp-streamify`, see [why here](http://stackoverflow.com/a/26127463/3995789). Finally since the output is minified, we change our output filename to `gulpp.min.js`.

We are done with our gulp tasks. Add a [script](https://docs.npmjs.com/misc/scripts) to `package.json` to clean build folder and compile your project into it.

```json
  "scripts": {
    "compile": "rm -rf build && gulp prod"
  },
```

Now run `npm run compile` and check out our bundled project inside `build` folder.

## Publish

Next step is to include this project inside another project, which we will call our main project. Although we can publish this to npm, we can also specify this project as a local dependency and use it locally.

Add [main entry point](https://docs.npmjs.com/files/package.json#main) to `package.json`. This is the output JavaScript file from your `gulp prod` task:

```json
  "main": "build/gulpp.min.js",
```

Run `npm link` in this project to specify it as a local module.

Next, go to our main project and install this project, when first time installing make sure you specify the local path to this project:

```
npm install ../gulpp --save
````

Now you can import and use [gulpp](https://github.com/eguneys/gulpp). Next time we update something on this project, we have to run `npm run compile` to re-build our project, and go to our main project and run `npm install gulpp` to re-install the updated version. Enjoy!
