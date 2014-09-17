---
layout: post
title: "Let's Build a Yeoman Generator 2"
date: 2014-09-17 12:14:58 +0300
comments: true
categories: nodejs yeoman phaser mocha tdd
tags: [nodejs, yeoman, phaser, mocha, tdd]
comments: true
image:
  feature: /images/diary-bg20.jpg
  background: colorflags-bg.jpg
---

## Introduction

In the previous [article](/blog/2014/09/17/lets-build-a-yeoman-generator-project-template-slash-w-gulp/) we built a Phaser project template, that used gulp as a build tool. In this article we will build a yeoman generator, for scaffolding, generating boilerplate from that template. We will also use mocha to write tests and see how to do test driven development. Finally we will publish it to npm.

## Initialize Yeoman Generator

A yeoman generator is an npm package, that runs on node environment. As we write our generator we will test it with mocha.

First install mocha with npm:

{% highlight bash %}
$ npm install -g mocha
{% endhighlight %}

Next we initialize npm, and create our `package.json` file.

{% highlight bash %}
$ npm init
{% endhighlight %}

Next, we install our dependencies:
{% highlight bash %}
$ npm install --save yeoman-generator yosay chalk chai fs-extra gulp mocha
{% endhighlight %}

We also need `yo` as a *peerDependency*, you can see the example [`package.json`](https://github.com/eguneys/generator-phaserjs/blob/master/package.json) as a reference, or just use it as is.

Next I've configured some utility modules, that are going to make our life easier, grab them as we will later use them. [`util.js`](https://github.com/eguneys/generator-phaserjs/blob/master/util.js), and [`test-util.js`](https://github.com/eguneys/generator-phaserjs/blob/master/test-util.js), place them in the root directory, next to `package.json`.


### Directory Structure

Here's the standard directory structure for yeoman generators.

{% highlight bash %}
.
|-- LICENSE
|-- README.md
|-- generators
|   |-- app
|   |   |-- templates
|   |   `-- index.js
|   |-- prefab
|   |   |-- templates
|   |   `-- index.js
|   `-- state
|       |-- templates
|       `-- index.js
|-- package.json
|-- test
|   |-- fixtures
|   |   |-- app.js
|   |   |-- bower.json
|   |   `-- package.json
|   |-- test-creation.js
|   |-- test-load.js
|   |-- test_prefab.js
|   `-- test_state.js
|-- test-util.js
`-- util.js
{% endhighlight %}

As you can see, besides `LICENSE`, `README.md`, `package.json` and utility files, we have two folders `generators` and `test`.

`generators` folder holds each sub-generator. Each folder is the same name as the sub-generator name. In our case we have a default generator `app`, that is called with command `yo name`, and two extra sub-generators `prefab` and `state`. Each sub-generator has an `index.js` file, that contains the entry point, and a `templates` folder that contains the template files for the boilerplate. Template files will use `underscore` templating library.

`test` folder holds the tests for the yeoman generator. `test/fixtures` folder holds fixture files that we need for some tests.

For now, create two folders `generators` and `test`, next we will write tests, and later write code to pass them.

## Write Failing Test, Code, Pass Test

That's how we develop our generator, using test driven development. Before we start, I want to tell you that we will write our tests in coffeescript. I never liked coffescript, until I found out the perfect opportunity to learn it, that is because it is more convenient for nested test code, you won't have to deal with nested brackets. Now let's write our first test.

### Default Generator `app`

Default generator used when you call `yo name`. It is the `app` generator, and it resides in the `app` folder.

`File: test/test-load.js`

{% highlight coffee %}
assert = require 'assert'

describe 'phaserjs generator', ->
  it 'can be imported', ->
    app = require '../generators/app'
    assert app != undefined
{% endhighlight %}

Here we use BDD interface that mocha provides, that is, `describe`, and `it` keywords. Mocha allows you to use any assertion library, we use `assert` module. `describe` defines a test suite, `it` defines a test case.

In our case we have a `phaserjs generator` test suite with a test that says `it can be imported`. In the test, we simply require our first generator, that is the main generator `../generators/app`, and assert that it is defined.

Now when you go to the root directory and run your tests with command `mocha`, it will fail, because we don't have an `app generator` yet, so let's code our first generator.

### Extending generator

Yeoman offers two base generators `Base` and `NamedBase` that we can extend from. For the default `app` generator we will extend `Base` generator.

`File: generators/app/index.js`

{% highlight js %}
var path = require('path');
var chalk = require('chalk');


// Two utility modules that I provided before
var util = require('util');
var genUtils = require('../../util.js');

// modules yeoman provides
var yeoman = require('yeoman-generator');
var yosay = require('yosay');

var PhaserGenerator = yeoman.generators.Base.extend({

});

module.exports = PhaserGenerator;
{% endhighlight %}

Here we extended the `Base` generator, and exported it as a module. Now the tests will pass.

But the generator doesn't do anything yet, now let's write some tests that asserts that our default generator actually creates some boilerplate.

`File: test/test-creation.js`

{% highlight coffee %}
path = require 'path'
fs = require 'fs-extra'
exec = require('child_process').exec

helpers = require('yeoman-generator').test
expect = require ('chai').expect

describe 'phaserjs generator', ->

  beforeEach (done) ->
    @app = helpers
      .run path.join(__dirname, '../generators/app')
      .inDir path.join(__dirname, '.tmp'), (dir) ->
      
        fs.symlinkSync path.join(__dirname, 'fixtures/node_modules'),
            path.join(dir, 'node_modules'), 'dir'

        fs.mkdirpSync path.join(dir, 'app/')

        // bower_components
        fs.symlinkSync path.join(__dirname, 'fixtures/bower_components'), path.join(dir, 'app/bower_components'), 'dir'

        done()

  describe 'running app', ->
    describe 'with default options', ->
      it 'should pass jshint', (done) ->
        @app.withPrompt({})
            .on 'end', ->
              console.log 'running jshint'
              exec 'gulp jshint', (error, stdout, stderr) ->
                if (error)
                  console.log 'Error: ' + error
                  
                expect(stdout).to.contain 'Finished \'jshint\''
                expect(stdout).to.not.contain 'problems'
                done();
{% endhighlight %}

First, we require `helpers` module, that yeoman provides as test helpers, and `chai` assertion library. Then we have a `beforeEach` block that runs before every test within the test suite. Before every test we have to run the generator, and then run our tests against the end results.

The way we run the generator is simple, thanks to `helpers` module. we provide our generator path to the `helpers.run` method. This will return a `RunContext` on which you can later setup directory, mock prompt, options etc.

After running our generator with `helpers.run` method, we use `inDir` method to setup the directory to run the generator from. We pass it a `.tmp` directory. The second parameter is a callback we use to setup the `.tmp` directory. In our case we symlink `node_modules` and `bower_components` we had in our `test/fixtures` folder beforehand, so we don't have to run `npm install` for our test scenario each time we test our generator.

The test will run `gulp jshint` and expect that it will finish with no problems.

To pass the test we need to generate the boilerplate Phaser project. Let's see how we do that next.

`File: generators/app/index.js`

{% highlight js %}
var PhaserGenerator = yeoman.generators.Base.extend({

  info: function() {
    this.log(this.yeoman);
    this.log(chalk.yellow(
      'Out of the box I create a Phaser app with RequireJS,\n' +
      'and gulp to build your app.\n'
    ));
  },

  generateBasic: function() {
        
    this.template('gitignore', '.gitignore');
    this.copy('jshintrc', '.jshintrc');
    this.copy('bowerrc', '.bowerrc');
    this.template('_package.json', 'package.json');
    this.copy('_bower.json', 'bower.json');
    this.template('_gulpfile.js', 'gulpfile.js');
  },

  generateClient: function() {
    this.sourceRoot(path.join(__dirname, 'templates'));
    
    genUtils.processDirectory(this, 'app', 'app');
    genUtils.processDirectory(this, 'build', 'build');
  },
    
  end: function() {
    this.installDependencies({
      skipInstall: this.options['skip-install']
    });
  }
});
{% endhighlight %}

Every method we added is called in sequence. First in the `info` method we display some ASCII art and info about our generator. Next in the `generateBasic` and `generateClient` methods we setup the boilerplate. And finally in the `end` method we run `installDependencies` method that runs `bower install && npm install` for us.

Note that there are three helper methods for copying files to setup the boilerplate. `this.copy` will simply copy the file from `generators/app/templates` folder to the users project folder, `this.template` will run the template engine, underscore, and copy the file. Finally we use `genUtils.processDirectory` that I added as a helper method that copies whole directories.

After coding our generator, we actually need template files to copy. They live in the `generators/app/templates` folder. Let's take a look at `_package.json` template and how we use underscore to customize the boilerplate files.

`File: generators/app/templates/_package.json`

{% highlight js %}
{
  "name": "<%= _.slugify(name) %>",
  "version": "0.0.0",
  "description": "Phaser template with requirejs and gulp",
  // ... rest is omitted
}
{% endhighlight %}

Here we use underscore templating, and helpers function to customize the name of the package. `name` variable is the name of the users project.

If you run the tests again, hopefully it will pass this time.

### Sub Generator `state`

Sub generator `state` used when you call `yo name:state`. It is the `state` generator, and it resides in the `state` folder.

`File: test/test_prefab.js`

{% highlight coffee %}
describe('phaserjs:prefab generator', ->
  defaultArgs = ['TestPrefab']

  beforeEach ->
    @app_prefab = helpers
      .run path.join(__dirname, '../generators/prefab')
      .inDir path.join(__dirname, '.tmp')

  describe 'with default options' ->
    it 'should generate expected files', (done) ->
      @app_prefab
        .withArguments {}
        .on 'end', ->
          assert.file ['app/scripts/prefabs/testprefab.js']

          assert.fileContent 'app/scripts/prefabs/testprefab.js', / function Testprefab\(game, x, y\) \{\n[\s\S]+Phaser\.Sprite\.call\(this, game, x, y, \'atlasname\', 0/

          assert.fileContent 'app/scripts/prefabs/testprefab.js', /Testprefab\.prototype = Object\.create\(Phaser\.Sprite\.prototype\)\;\n/
          
          assert.fileContent 'app/scripts/prefabs/testprefab.js', /Testprefab\.prototype.constructor = Testprefab;\n/

          done()

  describe 'with --group option', ->
    it 'should generate expected files', (done) ->
      @app_prefab.withOptions {'group':true}
                 .on 'end', ->

                   assert.fileContent 'app/scripts/prefabs/testprefab.js', / function Testprefab\(game, parent\) \{\n[\s\S]+Phaser\.Group\.call\(this, game, parent\)\;\n/

                   assert.fileContent 'app/scripts/prefabs/testprefab.js', /Testprefab\.prototype.constructor = Testprefab;\n/

                   done();
{% endhighlight %}

In this test suite, we have two tests, one uses the default options, second runs the generator with `--group` option. In each case we use `assert.fileContent` test helper to match a regex against the generated file.

To pass the test let's code our sub generator.

`File: generators/prefab/index.js`

{% highlight js %}
var PhaserJsStateGenerator = yeoman.generators.NamedBase.extend({
  init: function () {
    this.option('group', {
      desc: 'extend Phaser.Group',
      type: Boolean,
      defaults: false
    });
  },

  generate: function() {
    var slugName = this._.slugify(this.name),
        templateFile = 'prefab.js';
        
    if (this.options['group']) {
      templateFile = 'prefab_group.js';
    }
    this.template(templateFile, 'app/scripts/prefabs/' + slugName + '.js');
  }
});
{% endhighlight %}

Here we define an extra option `group` that can be passed to our sub generator. It defaults to false, you can enable it by calling the sub generator with group option `$ yo name:prefab --group`. In our case we use the option to decide which template we copy as boilerplate.

## Manually Test Your Generator

Now your generator is passing all the tests, yet you want to see it in action. Since you're developing the generator locally, and haven't published it yet, you have to symlink your local module as a global module, using npm.

Run this command in your generators root directory.

{% highlight bash %}
$ npm link
{% endhighlight %}

Now you can call `yo generator-name` and test your generator if it works.

## Publish Your Generator

Finally you can publish your generator. First create an account on [npm](http://npmjs.org). Next set your npm author info.

{% highlight bash %}
$ npm set init.author.name "Your Name"
$ npm set init.author.email "Your Email"
$ npm set init.author.url "Your Website"

$ npm adduser
{% endhighlight %}

Then publish with.

{% highlight bash %}
$ npm publish
{% endhighlight %}

### A note about the package.json format

There is a list of yeoman generators in the [official yeoman website](http://yeoman.io/generators/). It is automatically pulled from the npm API. To list your generator there you need to add `yeoman-generator` keyword to your `package.json`.

{% highlight js %}
{
  "name": "generator-phaserjs",
  "version": "0.1.0",
  "description": "Yeoman generator for Phaser with RequireJS",
  "repository": "eguneys/generator-phaserjs",
  "author": {
    "name": "Emre Guneyler",
    "email": "eguneys@hotmail.com",
    "url": "https://github.com/eguneys"
  },
  "keywords": [
    "yeoman-generator",
    "scaffold",
    "generator",
    "framework",
    "mvc",
    "phaser",
    "gulp",
    "requirejs"
  ],
  //.. rest is omitted
}
{% endhighlight %}

Before you publish remember these notes; set your package name properly, set a version number and bump it with every publish, set a description, set a github repository url, set your author details, set your keywords properly, and add a `yeoman-generator` keyword. Note that the repository field should fit either of these formats or it won't get listed in the yeoman website, `"repository": "eguneys/generator-phaserjs"`, `"repository": "git://github.com/eguneys/generator-phaserjs"`.

## Conclusion

In this article we have built a yeoman generator for Phaser using test driven development with mocha. We have learned how to write tests using BDD syntax, and use yeoman test helpers to ease testing, how to write yeoman generators using the yeoman generator API. Finally we have learned how to manually test our generator and publish it as a global npm module. You can find the full source code at [github](https://github.com/eguneys/generator-phaserjs). Note that you should always search similar generators and see their source code before you write your own and I recommend using our approach and building your project template first, as we did in the first article, then writing your generator will be greatly simplified.

There are more that we haven't covered you can do with yeoman API, such as user interaction, composability, storing user configuration and much more. For more information see the official yeoman website and these useful links.

* [generator list](http://yeoman.io/generators/).
* [API documentation](http://yeoman.github.io/generator/).
* [tuts+ tutorial](http://code.tutsplus.com/tutorials/build-your-own-yeoman-generator--cms-20040).
