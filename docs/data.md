# Data Management

*This chapter describes data management for external sources. If you are looking for using Data Sets in tests, see [Data Driven Tests](http://codecept.io/advanced/#data-drivern-tests) section*

Managing data for tests is always a tricky issue. How isolate data between tests, how to prepare data for different tests, etc.
There are different approaches to solve it:

1.  reset database completely between tests
2.  create unique non-intersecting data sets per each test
3.  create and delete data for a test

The most efficient way would be to allow test to control its data, i.e. the 3rd option.
However, accessing database directly is not a good idea as database vendor, schema and data are used by application internally and are out of scope of acceptance test.

Today all modern web applications have REST API. So it is a good idea to use it to create data for a test and delete it after.
API supposed to be a stable interface and it can be used by acceptance tests. CodeceptJS provides 2 helpers for Data Management via REST API.

## REST

[REST helper](http://codecept.io/helpers/REST/) uses [Unirest](http://unirest.io/nodejs.html) library to send HTTP requests to application.
However, it doesn't provide tools for testing APIs, so it should be paired with WebDriverIO, Nightmare or Protractor helpers for browser testing.

Enable REST helper in global config. It is recommended to set `endpoint`, a base URL for all API requests.
If you need some authorization you can optionally set default headers too.

See the sample config:

```js
"helpers": {
  "REST": {
    "endpoint": "http://localhost/api/v1/",
    "defaultHeaders": {
      "Auth": "11111"
    }
  },

  "WebDriverIO" : {
    "browser": "chrome"
  }
}
```

REST helper provides basic methods to send requests to application:

```js
I.sendGetRequest()
I.sendPostRequest()
I.sendPutRequest()
I.sendPatchRequest()
I.sendDeleteRequest()
```

As well as a method for setting headers: `haveRequestHeaders`.

Here is a usage example:

```js
let postId = null;

Scenario('check post page', function*(I) {
  // valid access token
  I.haveRequestHeaders({auth: '1111111'});
  // get the first user
  let user = yield I.sendGetRequest('/api/users/1');
  // create a post and save its Id
  postId = yield I.sendPostRequest('/api/posts', { author: user.id, body: 'some text' });
  // open browser page of new post
  I.amOnPage('/posts/2.html');
  I.see('some text', 'p.body');
});

// cleanup created data
After((I) => {
  I.sendDeleteRequest('/api/posts/'+postId);
});
```

This can also be used to emulate Ajax requests:

```js
I.sendPostRequest('/update-status', {}, { http_x_requested_with: 'xmlhttprequest' });
```

## Data Generation with Factories

This concept is extended by [ApiDataFactory](http://codecept.io/helpers/ApiDataFactory/) helper.
It builds data according to defined rules and uses API to store them and automatically clean them up after a test,
This way setting data for a test is as simple as writing:

```js
let post = yield I.have('post');
I.haveMultiple('comment', 5, { postId: post.id});
```

Just define how many items of any kind you need and ApiDataFactory will create them for you.
However, to make this work some preparations required.

At first, you need data generation libraries which are [Rosie](https://github.com/rosiejs/rosie) and [Faker](https://www.npmjs.com/package/faker). Faker can generate random names, emails, texts, and Rosie uses them
to generate objects using factories.

Install rosie and faker to create a first factory:

```js
npm i rosie faker --save-dev
```

Then create a module which will export a factory for an entity.

See the example providing a factory for User generation:

```js
// factories/post.js
var Factory = require('rosie').Factory;
var faker = require('faker');

module.exports = new Factory()
  .attr('name', () => faker.name.findName())
  .attr('email', () => faker.internet.email());
```

Next is to configure helper to match factories with API:

```js
 "ApiDataFactory": {
   "endpoint": "http://user.com/api",
   "factories": {
     "user": {
        "uri": "/users"
        "factory": "./factories/user"
     }
   }
 }
```

Then, calling `I.have('user')` inside a test will create a new user for you.
This is done by sending POST request to `/api/users` URL. Response is returned and can be used in tests.

At the end of a test ApiDataFactory will clean up created record for you. This is done by collecting
ids from crated records and running `DELETE /api/users/{id}` requests at the end of a test.
This rules can be customized in helper configuration.

---

### done()
