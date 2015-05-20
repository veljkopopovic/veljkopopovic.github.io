---
layout: post
title: "AngularJS"
date: 2015-04-21 17:00:00 +0100
comments: false
---
# AngularJS, version 1

Year and a half ago or so colleague of mine has came at the office: 'Have you seen AngularJS? Wow, it seems as a solution to lot of our problems with RT web pages ... Check it out, could be fun' ...
Nice docs, nice examples, hit the road, Jack ... Once initial research was done, real problems popped out: performance, code reuse, how to clean up and so on ...

So, let me share some of our experiences, solutions we used and how we managed to create relatively heavy duty pages we created for our costumers. Still, there is lot more to be done and hopefully version 2 will be even better.

Developer should use Angular modules for implementations. Modules represents a mechanism for registering, configuration, declaration and definition of different software constructs that will be used accross one or more apps. If you need more info, please consult [API docs](https://docs.angularjs.org/api/ng/type/angular.Module), details won't be explained here. Most interesting module components for this post are services, factories, controllers, values and constants. Let's see how they can make our life bit more easier.


### Services vs factories, are they the same or what?


**No, they are not the same thing.** Although, they do have one common property: **both providers are singletons.** In other words, code written as factory or service will be executed once and only once. [Even Angular docs did not make my life any easier](https://docs.angularjs.org/guide/services): guide name is **Services** and once I get on 'Creating Services' part, there is a ```.factory``` function call. On the other hand, there is an [API page](https://docs.angularjs.org/api/auto/service/$provide#service) which explains instantiation mechanism. Unfortunatelly, what .service call this actually does is an empty constructor registration which can be filled using ```this.some_field``` approach. This is not a good solution, and this approach should never be used due to certain performance drawbacks. Will be discussed bit later.

On the other hand, ```.factory``` call is very usefull. Whatever you do return from registered factory function, you will get once you use factory name as dependency.

```javascript
Angular.service ('MyClass', [function () {
  function MyClass() {
  }
  MyClass.prototype.method1 = function () {
  };
  MyClass.prototype.method2 = function () {
  };

  return MyClass;
}]);
```

Declaring a JS class this way, Angular injector will replace **MyClass** string in dependency list with constructor of a class. This is very usefull and will be discussed a bit leter.


### Abstractions and implementations


Any large scale programming will lead you to abstractions. So, where to put these within Angular system in a propper way? Ok, first things first: what is propper way for Angular? First of all, globals should be avoided as much as possible while writing Angular code. On the other side, try to use as much of Angular infrastructure to manage your code loading. So, we have to write some providers.
Two types of providers are suitable for abstractions:

- Angular constants: suitable for classes with no Angular dependency at all
- Angular factories: suitable for classes with some Angular dependency

Of course, constants and factories are suitable for implementations as well. Other providers should be used as implementations only.


### This is JavaScript, do I really need any classes and abstractions?


**Yes.**

JS is a dynamically typed programming language and no one can stop a programmer to add into object instance whatever he wants. Of course, there is a certain price to be paid. The most exausting explanation how V8 engine works has been written by Vyacheslav Egorov in this [article](http://mrale.ph/blog/2014/07/30/constructor-vs-objectcreate.html). This is an article about V8 engine, but there is no reason to assume that the other engines will handle random property addition any better. In short: it would be nice not to introduce new object properties once you created an object. So, if you have properties in your object, initialize them in constructor and use them afterwards, do not add them on the fly. More on that you can read in this [article](http://morethanslightly.com/index.php/2014/09/cleaning-up-the-objects-on-politeness/). Speaking of that, note that the most efficient way of adding properties to an object is inheritance using [```Object.create```](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/create) call.

In short: if you want to write some efficient code you'd be better do some very nice and polite abstractions and limit your key (property names) space to a finite set.

Speaking of abstractions let's take a look how Angular controllers work and reveal some problems with current Angular implementation.

*["In Angular, a Controller is a JavaScript constructor function that is used to augment the Angular Scope. When a Controller is attached to the DOM via the ng-controller directive, Angular will instantiate a new Controller object, using the specified Controller's constructor function. A new child scope will be available as an injectable parameter to the Controller's constructor function as $scope."](https://docs.angularjs.org/guide/controller). [This is a jsfiddle with very simple controller](http://jsfiddle.net/veljkopopovic/zc4thwwm/)*.



Several things should be noticed. First, we are adding properties to the $scope object on the fly. If we have one and only one controller on the page, thats not a problem at all. But if you have, let's say hunderds of them, JS engine created for each and every property added a hidden class. At the end, the page will end up with thousands of variants of $scope object. The page becomes slower and slower as amount of hidden classes rise, the JS engine optimisation won't do any good (what should I optimize if I don't know signature of an object, says a poor JS engine), memory consumption rises and there it goes. The price has been paid.

Obviously, giving a $scope object to controller developer to add properties on it was a bad idea, I hope version 2 will address this issue. What we can do? We can reduce amount of property names assigned to $scope object. Will be back on this topic in a minute.

Second important thing is: what about destruction? In an extreme situation, page will never be refreshed. Angular supports partials and that give us possibility to navigate through app with no page refresh at all. So, if DOM element which had a controller attached on was destroyed (for example, another partial was loaded), is there a way to clean the $scope object and release the unused memory? Yes there is. 

Every scope will raise an ```$destroy``` event. It's up to developer to do some clean up once raised. Extreme discipline is required to address this one correctly. At the glance, writing cleanup piece of code is very simple ([fiddle](http://jsfiddle.net/veljkopopovic/zc6dg1a0/)). But this solution has a small performance issue as well: each and every instance of the controller will create new instance of a doCleanUp function again and again.

Would it be nice to define a structure (class) with cleanUp method ([fiddle](http://jsfiddle.net/veljkopopovic/9c53omo8/)) and do this automatically with no memory hit (remember the price must be paid, but let's make it as small as possible)? Seems bit better. We automated destroy. If we wanted to inherit this class and do something from now on, solution is to put it global and there it goes. Let's analyze this piece of code. With this approach, $scope, actually, has one and only one field added on the fly: ```_ctrl```, in a very predictive manner. So we did manage to save JS engine leaving $scope object with one and only one property. On the other hand, only thing we lost is comfort, this approach requires bit more typing. Everything we wanted to implement in controller, we should put into extension of this basic class and in HTML, values and functions will have a _ctrl prefix all around. But, we did manage to reduce fieldset of $scope class and creating new classes will do some memory consumption within JS engine but with least possible amount of junk and with maximized optimization. So far so good.

What about basic class and how to expose it? Globaly? If you do have several types of controllers, all of them will inherit SimpleControllerClass in similar way and we end up with bunch of global classes all around our code. Problems becomes more obvious if we wanted to use some of Angular providers with this class. So, make an Angular abstraction like something like this:

```javascript
angular.module('myApp', [])
  .factory('SimpleControllerClass', [function () {
    function SimpleControllerClass($scope) {
      $scope._ctrl = this;
      this.$scope = $scope;
      $scope.$on('$destroy', this.doCleanUp.bind(this));
    }
    SimpleControllerClass.prototype.doCleanUp = function () {
      this.$scope = null;
      $scope._ctrl = null;
    };
    return SimpleControllerClass;
  }])
  .factory('IntervalClass', ['SimpleControllerClass', '$interval', function (SimpleControllerClass, $interval) {
    function IntervalClass($scope) {
      SimpleControllerClass.call(this, $scope);
        this.interval = $interval(this.up.bind(this), 1000);
        this.val = 0;
      }
      IntervalClass.prototype = Object.create(SimpleControllerClass.prototype);

      IntervalClass.prototype.up = function () {
        this.val++;
      };

      IntervalClass.prototype.doCleanUp = function () {
        this.interval.cancel();
        this.interval = null;
        this.val = null;
        SimpleControllerClass.prototype.doCleanUp.call(this);
      };
   }])
  .controller('IntervalController', ['$scope', 'IntervalClass', function ($scope, IntervalClass) {
    new IntervalClass($scope);
  }]);
```
[fiddle](http://jsfiddle.net/veljkopopovic/gyypas36/)

Let's take a minute to analyze this piece of code. Why factory? This way we did not create a global class, we created an Angular friendly one. ```IntervalClass``` implments ```SimpleControllerClass```. Since we needed an ```$interval``` Angular service, it is very comfortable to use factory to define ```IntervalClass```. Avoiding globals and making Angular friendly classes, we used every possible advantage Angular gave us regarding code organisation (dependencies management and injection).
Every dependency reqiured by ```IntervalClass``` is written in ```IntervalClass``` factory definition, so all controller needs is ```IntervalClass``` and ```$scope```. Note ```doCleanUp``` method: we cleanly shut down interval instance([Angular docs note: **Intervals created by this service must be explicitly destroyed when you are finished with them.**](https://docs.angularjs.org/api/ng/service/$interval))


### Conclusion


So let's sum up what have we done so far. Issues we were dealing within this post was
- AngularJS friendly abstraction basics, what we need from Angular and what Angular can give to us
- reducing memory bloat produced by controllers using abstraction mechanism

Abstractions has been introduced via Angular constants and factories. Main goal was to avoid multiple code execution: write abstract code once and execute it once, instantiate as much as you like. Memory bloat was avoided using reduced proprties set applied to $scope object. On the other hand, bit of more typing is required in markup, typing _ctrl prefix.

Will be back soon with some benchmarks.
