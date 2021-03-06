# Cucumberjs example
Just a simple implementation of
[cucumberjs](https://github.com/cucumber/cucumber-js) with
[phantomjs](http://phantomjs.org/), [chai](http://chaijs.com), and
[webdriverio](http://webdriver.io).

## Usage
* `npm install` to install dependencies
* `./node_modules/.bin/phantomjs --webdriver=4444` to start phantomjs
* `./node_modules/.bin/wdio` to run tests. It will use the `wdio.conf.js` config file by default.
* `./node_modules/.bin/cucumber-js` will fail without the webdriver wrapper config but if you have steps in your features that aren't defined it will spit out boilerplate code to drop into your step definitions. Handy! (This is the main reason I prefer cucumber to [Yadda](https://github.com/acuminous/yadda))

## Usage in other projects
* `npm init` and hit enter a bunch
* `npm install --save-dev cucumber phantomjs webdriverio chai`
* `./node_modules/.bin/wdio config` to walk through a wizard and define your
  webdriverio config. Sweet!
 * choose cucumber for the framework aka test runner
 * set the reporter to spec
 * set the logging verbosity to verbose
 * set the base url accordingly
* create a `features` directory and write
  [gherkin](http://docs.behat.org/en/latest/guides/1.gherkin.html) features
  there such as `features/MyCoolFeature.feature`
* run `./node_modules/.bin/cucumber-js` It will check your feature files for step definitions that you don't have code for and   spit out example snippets for those. You can put them in
  `features/step-definitions/Whatever.js`. Each definition file needs to contain
  an exported function via module.exports, CommonJs style. Put snippets in that
  function. ([example
  here](https://github.com/mikedfunk/cucumberjs-example/blob/master/features/step-definitions/GoogleTitleTestSpec.js))
* add a [cucumber world
  file](https://github.com/mikedfunk/cucumberjs-example/blob/master/features/support/world.js)
  in `features/support/world.js` to set up chai in all tests. If you want. Or if you have one step definitions file you can      just define your dependencies at the top.
* Replace  `callback.pending();` calls with [webdriverio
  api calls](http://webdriver.io/api.html) to go to urls, click things, etc. By running cucumber through wdio
  you get `browser` defined as a global, so you can just call
  `browser.url('sub-url here').then(...)`,
  `browser.getText('selector').then(...)`, etc. inside your spec definitions. ([full api docs](http://webdriver.io/api.html))
* Replace `callback` in the function params with `next` or `done`. It makes a lot more
  sense.
* Assert with `this.expect(value).to.equal(expected);`. Chai has [other
  tests](http://chaijs.com/api/) too. When a test fails, it will throw an Error
  and stop that step. Call `next()` after all expectations are set. The only
  way I could get next working in promises was via `.then(...).call(next);`. If
  all passes it will continue to the next step definition.
* start phantomjs and run tests per usage instructions above.
* `./node_modules/.bin/wdio` to run tests. `./node_modules/.bin/cucumber-js` will not actually run tests directly because it is
  not configured directly, it's wrapped by webdriverio.

## Gotchas
In a step definition this will cause trouble because the next step will try to
execute whether or not the browser has finished loading the page. You will
probably encounter a race condition. Instead, use the `.then()` method:

```javascript
module.exports = function() {

  this.When(/^I visit the homepage$/, function (next) {
  
    // bad!
    browser.url('/');
    next();

    // good!
    browser.url('/').then(function () {
      next();
    }).catch(function (err) {
      next(err);
    });
    
  });
};
```

`this` from `world.js` is available in each step definition, but once you get
to a promise or callback from the webdriverio api `this` is something different.
Fix it with the `.bind(this)` part below:
```javascript
// option 1
browser.getText('form[name="register"]').then(function(text) {
  // `this` refers to the parent context now because of the `.bind()` below
  this.expect(text).to.not.be.empty;
  next();
}.bind(this)).catch(function (err) {
  next.err();
});

// option 2
// this is useful when you go several layers deep in functions and
// you don't want to keep calling `.bind()` for each one
var _this = this;
browser.getText('form[name="register"]').then(function(text) {
  _this.expect(text).to.not.be.empty;
  next();
}).catch(function (err) {
  next.err();
});
```

---------------------------------------

Don't call `next()` twice in a step! It will confuse cucumber and confuse the shit out of you because steps will be firing out of sequence, even in the wrong scenarios. Took me forever to find this.

---------------------------------------

The [`yield` ES6 generator method](https://github.com/webdriverio/webdriverio/blob/master/examples/runner-specs/jasmine.spec.js) of using the webdriverio API doesn't work unless you compile down with babel/traceur.
