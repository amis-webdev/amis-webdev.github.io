---
title: Automatic error handling in AngularJS
author: paco
---
In demos and tutorials we only care about the happy-flow of our applications. That is ok, because we only run that code in our own little predictable environment and leaving everything else out makes any framework look nice and easy. Often in blog posts *"error handling and such is left as an exercise to the reader"* (which of course is an exercise nobody needs). A problem with good error handling code is that it clutters your "happy-flow code", which then becomes harder to read, and is quite a lot of work to get right. Wouldn't it be great if we could just write our happy code and get user friendly error messages for free? Is it possible to have some kind of automatic error handling in AngularJS?


The AngularJS project I worked on recently already had a component in place to show good-looking error messages to the user, all in accordance with their corporate identity and UX/UI design (comparable to [toastr](https://github.com/CodeSeven/toastr)). That component didn't do very much without having actual error messages to show, but being a lazy developer (who isn't?) I did not like the prospect of having to go through all our code and add handling code all over the place. So I started looking for a super easy, automatic, generic, pragmatic, intelligent, lazy solution.

# $exceptionHandler

A seemingly obvious thing to do is to replace the built-in [$exceptionHandler](https://code.angularjs.org/1.2.26/docs/api/ng/service/$exceptionHandler) service to catch any uncaught error, extract the message and send that to the error messages component. I was happy with that solution for about 2 seconds, until I realised it had two obvious flaws:

1. The error messages it produced were "not really user friendly"
2. A lot of errors never get to the `$exceptionHandler`!

**Flaw 1:** Error messages should ideally provide three pieces of information to the user: what went wrong, what was the application trying to do at that time and what can the user do to fix it (if possible)? For example, the message: *"An unknown error occurred at the server"* doesn't say much. It would be better to say: *"Unable to get the list of documents. An unknown error occurred at the server. Please try again in a few minutes"*. With the `$exceptionHandler` we get a message that says "what went wrong", but all context is missing (actually, we get a stack-trace, but that is not something an ordinary user is interested in)

**Flaw 2:** Most of the more interesting operations in data-intensive web applications are asynchronous, including all HTTP calls to the server. In AngularJS asynchronous operations work with [promises](https://code.angularjs.org/1.2.26/docs/api/ng/service/$q). With promises errors are not thrown and caught. You need to attach callback functions that handle errors (for more information on this mechanism [read this](https://github.com/kriskowal/q)).

OK, so that didn't work out. Let's take a step back and first implement good error messages manually. Look at [the companion page][companion] for demos of all scenarios.

# The hard way

In our case the manual error handling has to be done in the controller, because we don't want the services to have any knowledge of the error-messages component (especially in case of built-in or third-party services). So what does that look like? Say, we have the following happy-flow code:

```javascript
angular.module('ExampleModule', [])
  // ExampleController is a controller that enables to load some additional data after the press of a button.
  // This data is then accessible on $scope.data.
  .controller('ExampleController', function ($scope, someService) {

    $scope.loadAdditionalData = function () {
      someService.loadIt().then(function (data) {
        $scope.data = data;
      });
    };

  });
```
*(ExampleController without error handling)*

We want to:

1. Catch exceptions
2. Catch asynchronous errors
3. Provide a user-friendly error message to the user including meaningful context.

We get something like this:

```javascript
angular.module('ExampleModule', [])
  // ExampleController is a controller that enables to load some additional data after the press of a button.
  // This data is then accessible on $scope.data.
  .controller('ExampleController', function ($scope, someService, showErrorMessage) {

    $scope.loadAdditionalData = function () {

      var context = 'load additional data';

      try {

        someService.loadIt()
          .then(function (data) {
            $scope.data = data;
          })
          .catch(function (err) {
            // Catch promise rejections (e.g. problems with HTTP call)
            showErrorMessage(err, context);
          });

      catch (err) {
      	// Problems in our loadIt() synchronous code.
      	showErrorMessage(err, context);
      }
    };

  });
```
*(ExampleController with error handling)*

As you can see, we now have more error handling code than "happy code", but we do have all bases covered (assuming we have a slightly intelligent `showErrorMessage` service that extracts error messages from Errors and uses the context for better messages, etc). So how do we go from here? We could put the error handling code in our service to unclutter the controller, but that just moves the problem to a different place, it doesn't solve anything. Besides, some services are built-in or third-party, we cannot change those. So we have to do something else. Let's start by putting the boilerplate in a separate service.

# Step 1: Remove boilerplate

The first step is to take all this boilerplate code, put it in a service and handle all error cases automatically with one call. So we need a service (`errorHandler`) that can do the following:

```javascript
angular.module('ExampleModule', [])
  // ExampleController is a controller that enables to load some additional data after the press of a button.
  // This data is then accessible on $scope.data.
  .controller('ExampleController', function ($scope, someService, errorHandler) {

    $scope.loadAdditionalData = function () {

      // The method [errorhandler.call(method, self, context, args...)] handles all error cases
      // and returns whatever the method returns.
      errorHandler.call(someService.loadIt, someService, 'load additional data')
        .then(function (data) {
          $scope.data = data;
        });

    };

  });
```
*(ExampleController with less boilerplate)*

This removes a lot of boilerplate. It doesn't do much for readability and we still need to provide the context ('load additional data') everywhere we use any service, but it is a start. Look at [the companion page for a working demo][companion].

# Step 2: Decorate your services

The next step is to completely remove all boilerplate and return to our "happy code" (the first code fragment). How do we do that? AngularJS provides a way to decorate services with your own code (using: [$provide.decorator()](https://code.angularjs.org/1.2.26/docs/api/auto/service/$provide#decorator)). We can use that to automatically wrap all function-calls in an `errorHandler.call` call. Our complete example code now looks like this (including the `someService`):

```javascript
angular.module('ExampleModule', ['OurErrorHandlerModule'])

  // Our [someService] does a simple HTTP call
  .factory('someService', function ($http) {
    
    // The code is simple, but a lot can go wrong here...
    function loadIt() {
      return $http.get('somefile.json')
        .then(function (result) {
          return result.data;
        });
    }

    // We can provide the context inside the service if it is our own service, it serves as documentation as well,
    // you can also provide the context seperately in a .run(...) block. This is the only extra thing we have to do
    // for our services.
    loadIt.description = 'load additional data';

    // The actual service:
    return {
      loadIt: loadIt
    }
  })

  // Decorate the service. Because we can do this outside of the service, we can also use this for built-in or third-party services.
  .config(function (errorHandlerProvider, $provide) {
    errorHandlerProvider.decorate($provide, ['someService']);
  })

  // ExampleController is simple again... but this time with complete error handling.
  .controller('ExampleController', function ($scope, someService) {

    $scope.loadAdditionalData = function () {
      someService.loadIt()
        .then(function (data) {
          $scope.data = data;
        });
    };

  });
```
*(Example code using the complete errorHandler service)*

The source code of this errorHandler service is available in [this GitHub repository][repo]. As you see, the only step we need to take for complete error handling is the .config(…) block above. If you want to provide extra context for the user, you can simply provide a description on your functions and voila, you're done. For a demo and more information you can look at [the sourcecode][repo] and [the demo page][companion].

# PS: Zones

In the future <https://github.com/btford/zone.js/> might just provide a better way (or at least way cooler) to get a context for your messages (already available in Dart: <https://www.dartlang.org/articles/zones/>).

[companion]: https://pavadeli.github.io/angularjs-errorhandling/
[repo]: https://github.com/pavadeli/angularjs-errorhandling
