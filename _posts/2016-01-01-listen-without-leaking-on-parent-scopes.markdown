---
layout: post
title:  "Listen without Leaking on Parent Scopes"
date:   2016-01-10 00:00:00
categories: [angular, js]
---

# The Problem

Sometimes a controller needs to listen to events on ancestor scope.
Consider the case where controller A wants to listen to events from controller B.
Each controller is on a different branch in the scope hierarchy so events can not propagate from controller B to A.

Why? Events can be sent "down" to child scopes using `$broadcast()` and they can be sent "up" to parent scopes using `$emit()` but there's no way to to have an event first traverse "up" and then back "down" to sibling scopes.

What controller A must do is listen on an ancestor scope that is shared with controller B; as an example — `$rootScope`, which will always be a shared ancestor of any two scopes.

*Note: communication between controllers can also be accomplished using a shared service but that is not always going to be feasible*

# Some Background

Listening to events on an Angular scope is as simple as:

```javascript
$scope.$on('someEvent', function () {
  ...
});
```

Behind the scenes, Angular takes care of the [listener cleanup](https://github.com/angular/angular.js/blob/v1.4.10/src/ng/rootScope.js#L909) when the scope is destroyed:

```javascript
for (var eventName in this.$$listenerCount) {
  decrementListenerCount(this, this.$$listenerCount[eventName], eventName);
}
```

...which, in user space, is the equivalent of:

```javascript
  var unregisterHandler = $scope.$on('someEvent', function () {
    ...
  });
  $scope.$on('$destroy', unregisterHandler);
```

`$scope.$on()` returns an unregister function for the listener, just like Javascript's `setTimeout()` or `setInterval()`.
Then, in the event handler for the scope's `$destroy` event, the unregister function is called.

# A Catch...

Getting back to the original problem: what if a controller needs to listen to events on a parent scope?

```javascript
$rootScope.$on('someEvent', function () {
  ...
});
```

Simple, right? Except for one thing: the root scope does not get destroyed until the app shuts down.
So while your controller may be created and destroyed multiple times those event handlers are registered with each controller creation but are not removed until the app ends.

# A Solution Appears!

We need to replicate what Angular implicitly does during a scope's destruction.
To make this functionality reusable, we can to expose it as part of a `Toolbox` singleton — a service — that handles the listener cleanup:

```javascript
myApp.service('Toolbox', ['$rootScope', function ($rootScope) {

  return {

    AddParentscopeListener: function AddParentscopeListener ($scope, eventName, handler, $parentScope) {
      // Use parent scope if passed in, otherwise use root scope
      $parentScope = $parentScope || $rootScope;
      // Add the listener and save the returned unregister function
      var unregisterHandler = $parentScope.$on(eventName, handler);
      // Hook into the current scope's $destroy event
      $scope.$on('$destroy', function () {
        try {
          unregisterHandler();
        } catch (ex) {
          // Something weird happened, to log it
          console.log('Failed to unregister rootScope event handler', {
            error: ex,
            eventName: eventName
          });
        }
      });
    }

  };

}]);
```

Example usage:

```javascript
myApp.controller('ExampleController', ['$scope', 'Toolbox', function ($scope, Toolbox) {
  Toolbox.AddParentscopeListener('someEvent', function (evt) {
    ...
  });
}]);
```

Now, `ExampleController` will listen on `someEvent` events and will stop listening as soon as the controller is destroyed.
