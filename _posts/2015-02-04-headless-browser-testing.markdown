---
layout: post
title: "Automated Acceptance Testing with Headless Browsers"
date: 2015-02-04 13:00:00
owner: Daniel, Marian, Reto
tags: [testing,browser]
---

Test various testing frameworks and headless browsers to write automated *integration tests* of web applications.

<!--more-->

The test case we want to run for [denkmal.org](https://github.com/denkmal/denkmal.org/):

1. Open http://www.denkmal.dev/
2. Click "Add Event" in the navigation
3. Fill out form
4. Click "Submit" button
5. Expect "Thank you"

### [CasperJS](http://casperjs.org/)
Testing framework based on *PhantomJS*, written in JavaScript.
Generally feels mature, good docu, lots of resources on the Internet.

#### Installation:
```
brew install phantomjs192
brew install casperjs --devel
```

#### Test case:
```javascript
casper.test.begin('Can navigate to "add" page', function suite(test) {
  casper.start('http://www.denkmal.dev/');

  casper.thenClick('a.addButton', function() {
    this.waitForSelector('.Denkmal_Page_Add');
  });

  casper.then(function() {
    test.assertExists('.Denkmal_Form_EventAdd');
  });

  casper.run(function() {
    casper.capture('screenshot.png');
    test.done();
  });
});

casper.test.begin('Can submit a new event', function suite(test) {
  casper.start('http://www.denkmal.dev/add');

  casper.then(function() {
    var dateFrom = new Date(new Date().getTime() + 86400);
    dateFrom.setHours(21, 30);

    this.fill('form.Denkmal_Form_EventAdd', {
      'venue': 'My Venue',
      'venueAddress': 'My Address 1',
      'venueUrl': 'http://www.example.com/',
      'date[year]': dateFrom.getFullYear() + 1,
      'date[month]': dateFrom.getMonth() + 1,
      'date[day]': dateFrom.getDate() + 1,
      'fromTime': dateFrom.getHours() + ':' + dateFrom.getMinutes(),
      'title': 'My Title'
    }, false);
  });

  casper.waitForSelector('.Denkmal_Component_EventPreview', function() {
    test.assertSelectorHasText('.Denkmal_Component_EventPreview .event-location', 'My Venue');
    test.assertSelectorHasText('.Denkmal_Component_EventPreview .event-details', 'My Title');
    test.assertMatch(casper.fetchText('.Denkmal_Component_EventPreview .time'), /21:30/);
  });

  casper.thenClick('button[value="Hinzufügen"]', function() {
    casper.waitFor(function() {
      return casper.evaluate(function() {
        return __utils__.visible('.formSuccess');
      });
    });
    casper.wait(500);
  });

  casper.then(function() {
    test.assertVisible('.formSuccess');
    test.assertNotVisible('.Denkmal_Form_EventAdd .preview');
    test.assertNotVisible('.Denkmal_Form_EventAdd .formWrapper');
  });

  casper.run(function() {
    test.done();
  });
});
```

#### Run test case:
```
casperjs test my_test.js
```

### [WebSpecter](https://github.com/jgonera/webspecter)
BDD-style testing framework for *PhantomJS*. Focuses on concise syntax (with CoffeeScript).
Not much docu and not maintained any more. Difficult to debug and develop tests because of limited API.

#### Installation:
```
git clone https://github.com/jgonera/webspecter.git --recursive
```

#### Test case:
```coffeescript
feature 'Landing page', (context, browser, $) ->
  before (done) -> browser.visit 'http://www.denkmal.dev', done

  it 'has working "add" button', (done) ->
    $('a.addButton').click()
    wait.until $('.Denkmal_Page_Add').is.present, for: 2000, ->
      # Three different Chai assertion styles: expect, should, assert
      expect($('.Denkmal_Form_EventAdd').present).to.be.true
      $('.Denkmal_Form_EventAdd').present.should.be.true
      assert.isTrue($('.Denkmal_Form_EventAdd').present)

      done()

feature 'Event add page', (context, browser, $) ->
  before (done) -> browser.visit 'http://www.denkmal.dev/add', done

  it 'can submit a new event', (done) ->
    dateFrom = new Date(new Date().getTime() + 86400);
    dateFrom.setHours(21, 30);

    $('input[name="venue"]').fill 'My Venue'
    $('input[name="venueAddress"]').fill 'My Address 1'
    $('input[name="venueUrl"]').fill 'http://www.example.com/'
    $('select[name="date[year]"]').fill dateFrom.getFullYear() + 1
    $('select[name="date[month]"]').fill dateFrom.getMonth() + 1
    $('select[name="date[day]"]').fill dateFrom.getDate() + 1
    $('input[name="fromTime"]').fill dateFrom.getHours() + ':' + dateFrom.getMinutes()
    $('input[name="title"]').fill 'My Title'

    $('button[value="Hinzufügen"]').click()

    wait.until $('.formSuccess').is.visible, for: 2000, ->
      done()
```

#### Run test case:
```
webspecter/bin/webspecter my_test.coffee
```

### [DalekJS](http://dalekjs.com/)
Cross-browser testing framework using *PhantomJS*, *Chrome*, *Firefox* etc.
New tool on the block, still under development. Good documentation.

#### Installation:
```
npm install dalek-cli -g
npm install dalekjs --save-dev
```

#### Test case:
```javascript
module.exports = {
    'Can navigate to `add` page': function (test) {
        test.open('http://www.denkmal.dev')
            .screenshot('homeDenkmal.png')
            .assert.exists('a.addButton', 'Event hinzufügen link exists')
            .click('a.addButton')
            .waitFor(function () {
                return !$.active;
            })
            .assert.url().is('http://www.denkmal.dev/add', 'We are in the `add` page')
            .assert.exists('form.Denkmal_Form_EventAdd')
            .screenshot('addEventForm.png')
            .done();
    },

    'Can submit a new event': function (test) {
        test.open('http://www.denkmal.dev/add')
            .type('#s2id_autogen2', 'Dalek venue')
            .waitForElement('.select2-highlighted', 10000)
            .click('.select2-highlighted')
            .type('input[name="venueAddress"]', 'Address of the venue')
            .type('input[name="venueUrl"]', 'http://www.acme.com')
            .setValue('input[name="fromTime"]', '21:00')
            .type('input[name="untilTime"]', '23:00')
            .type('input[name="title"]', 'This is a test event')
            .type('input[name="artists"]', 'Cargo Media band')
            .type('input[name="genres"]', 'Heavy metal')
            .type('input[name="urls"]', 'http://www.cargomedia.ch')
            .submit('form.Denkmal_Form_EventAdd')
            .waitFor(function () {
                return !$.active;
            })
            .wait(500)
            .screenshot('formSubmitted.png')
            .assert.visible('.formSuccess')
            .done();
    }
};
```

#### Run test case:
```
dalek my_test.js
```

### Summary
We liked DalekJS the most, because of the simple yet powerful API, good documentation and cross-browser support.
Next step: Run web server and tests in CI.

Because our web applications are usually written with PHP, it might make sense to look for a PhantomJS-based tool for PHP.
This would allow us to run our application code in `setUp` and `tearDown` to create specific test environments.
For example: [PHP PhantomJS](http://jonnnnyw.github.io/php-phantomjs/) (PhantomJS binding only), [Codeception](http://codeception.com/) (acceptance testing framework).

**UPDATE:**

A followup about [using PhantomJS with PHP](/2015/03/04/phantomjs-php.html) was published.
