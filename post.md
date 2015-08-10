Synopsis

> Understanding scope is a vital part of writing robust Angular directives. It's one of those concepts that seems simple at first, but turns out to have some nuances that can make or break your app -- especially when transclusion comes into the mix or you begin to build directives on top of one another. 

>This post will delve into some of these scope-related nuances by explaining the scope hierarchy of an example application.  It assumes some prior knowledge of Angular directives and the Directive Definition Object (DDO), so if you're brand new at directives, I'd suggest taking a look at the documentation first<sup>[Documentation for Angular 1.x directive API](https://docs.angularjs.org/api/ng/service/$compile)</sup>.

Let's start with some basics.

## What is directive scope?

Typically, when developers refer to a directive's scope, they mean the scope bound to that directive's template during the linking phase.  This scope -- configurable through the `scope` property in the DDO -- is the execution context that Angular uses to look up any expressions defined in the template (such as {{ }} bindings).

There are three types of scope that you can configure for your directive's template:

1. shared scope (`scope: false`)

2. child scope (`scope: true`)

3. isolate scope (`scope: {}`)

For most directives, the template's scope is all you need to consider. However, it's important to note that directives that transclude<sup>[Documentation for transclusion](https://docs.angularjs.org/api/ng/service/$compile#transclusion)</sup> have a second scope in addition to the template scope: a transclusion scope that follows different rules. But more on that in a moment.

Given that brief introduction, let's dive into an example to dig into these concepts.

## Scope example app

Let's say we are building an application with the help of two components:

1.  a site-layout component called `ot-site`
2.  a list component called `ot-list`

![Example app](https://i.imgur.com/fYBhOLv.png)

`ot-site` provides the UI scaffold for our application: the static header, logo, and footer that will appear on every page. However, like any layout component, it has to allow the directive user to pass in some arbitrary, dynamic content for the body section, because that will inevitably change from page-to-page. Given these two requirements,  `ot-site` is a classic use case for transclusion.  Through `ot-site`, we can review how transclusion scope works*.

`ot-list` is a simple list component that takes a set of data and turns it into a styled `ul` element with selection logic.  We can use `ot-list` to demonstrate the three different options for passing data through into the template scope and how they affect the directive.

**This layout component and transclusion in general is discussed in more detail in a [previous blog post](https://www.airpair.com/angularjs/posts/creating-container-components-part-2-angular-1-directives).  Here, the implementation of `ot-site` is simplified, as our focus is on scope.*


## Transclusion scope

We'll start with`ot-site`. Before jumping into its scope hierarchy, let's take a moment to review how it is structured.

### Brief intro to transclusion

As mentioned before, we'd like to pass arbitrary content into the body of our scaffold, and the best way to do this is through transclusion.  If we make `ot-site` a transcluding directive, any HTML passed between its opening and closing tags will be transcluded into the template.  So if we want `ot-list` to appear in the body of the layout scaffold, our application markup might look like this:

**index.html **

```markup
<ot-site>
  <ot-list></ot-list>
</ot-site>
```

To hook up the transclusion on the JS side, our directive definition just needs to:

* add a template with its default scaffold (header, logo, etc)
* set `transclude` to true (to let Angular know to save the content between the directive tags)
* and add the `ng-transclude` directive to the element where the content should go.


**otSite.js**
```javascript
angular.module("ot-components")

.directive("otSite", function() {
  return {
    transclude: true,
    scope: {},
    template: [
      '<div class="ot-site">',
        '<div class="ot-site--head">',
          '<span class="ot-site--logo"></span>',
          '<h1 ng-bind="title"></h1>',
        '</div>',
        '<div class="ot-site--menu"></div>',
        '<div class="ot-site--body" ng-transclude></div>',
        '<div class="ot-site--foot">',
          '&copy; 2015 OpenTable, Inc.',
        '</div>',
      '</div>'
    ].join('')
  };
});
```

So given that implementation, where does transclusion scope come in?

### Template scope vs. transclusion scope

As a transcluding directive, `ot-site` essentially has two templates:
1. its own internal template (header/logo/footer defined in `otSite.js`)
2. the custom HTML passed in by the directive user (`ot-list` tags in `index.html`)

Because the templates are eventually combined in the DOM when the custom HTML is appended to the directive template, it's intuitive to assume that the two pieces will share the same scope once they are brought together.  But in fact, they are completely separate scopes that follow different rules.

1. The internal template's scope is always controlled by whatever you set in the `scope` property of the DDO, as described earlier.

2. However, the scope of the custom HTML (a.k.a. the transclusion scope) is unaffected by how you've configured the `scope` property. **It will always be a child scope of whatever outer context the directive was placed in.** **

***If you don't use the built-in transclusion functionality and transclude manually (by using the low-level transclude function), you can technically pass in whichever scope you'd like to be linked to the custom template. However, this is NOT recommended because it typically breaks bindings.*

### Wait, what's a child scope?

Let's take a step back for a moment. 

Scopes in Angular are organized into a hierarchy. When you bootstrap your application with `ng-app`, exactly one `rootScope` is created to form the top of that hierarchy.   As the root of the scope tree, it's the scope from which any other scopes created in your application descend. 

```
       rootScope
    child    child
child            child
```

So when a new scope is created (for example, by a directive like `ng-controller` or maybe one of your own), it becomes a child scope of the root scope or one of its descendants. 

Child scopes in the hierarchy inherit prototypically from their parent scopes, all the way up to the `rootScope`. When a lookup fails for a property on the child scope, next it will check the parent's scope for that property, and so on up the chain.

This hierarchy broadly mimics the DOM structure of the app.  Which scope is bound to a particular HTML tag does depend on where the tag falls in the DOM (with some notable exceptions in directives with isolate scope).  So if a tag is within a `div` that contains an `ng-controller` (which creates its own child scope), that tag will be within that controller scope's sphere of influence.

### The current scope hierarchy

Let's zoom out and take a look at where the `ot-site` tag has been placed in `index.html`, so we can start to understand the scope hierarchy.

**index.html**

```markup
<html ng-app="ot-components">
  <head>...</head>
  <body ng-controller="AppController">
    <ot-site>
      <ot-list></ot-list>
    </ot-site>
  </body>
</html>
```

From the broader `index.html`, we can see that there are two higher-level scopes defined in this application so far:

1. **the rootScope** - which stems from whichever element `ng-app` is on (`<html>`).
2. **the AppController scope** - which stems from whichever element has the matching `ng-controller` tag (`<body>`). This is a child scope of the `$rootScope`.

So where does the scope of `ot-site`'s custom template -- the transclusion scope -- fall in this hierarchy?  As mentioned, the transclusion scope is always a child scope of its outer context.  Because `<ot-site>` has been placed within the `<body>` tags, the `AppController` scope is its outer context.  Thus, the transclusion scope for `ot-site` will be a child of the `AppController` scope.

** scope hierarchy **

![Scope hierarchy](https://i.imgur.com/5UBVNmL.png)

As a child of `AppController`, the custom template is perfectly set up to inherit any bindings it needs from the broader application. This makes sense for transclusion, because if you had to pass in each model to the directive explicitly, it wouldn't truly support arbitrary content. The directive itself would have to anticipate every potential toggle or piece of data, which has its limits.


So where does the actual `ot-site` template (the `divs` that represent the header and logo, etc) fall in that tree? 

You may have noticed that we set an isolate scope for `ot-site`earlier (`scope: {}`), so unlike the transclusion scope, the scope for the template does **not** inherit prototypically from anything:

![Full scope hierarchy](https://i.imgur.com/F8Cwe4i.png)

As such, it's removed from the prototype chain.  While the custom template can reach up to access bindings from the controller, the isolated template is protected from any leaking to or from the application (more on this later).

Now that we understand `ot-site`, let's take a look at `ot-list`.

## Template scope

The template  for `ot-list` is fairly straightforward. Basically, we're just iterating over a list of items with an `ng-repeat` and setting up a selection callback:

**ot-list.html**
```markup
<ul>
    <li ng-repeat="item in items" ng-bind="item" 
      ng-class="{'ot-selected': item === selected}"
      ng-click="selectItem(item)">
    </li>
</ul>
```

From the template, you can see that we really need two pieces of information to generate our `ul`: 

1.  The data set (`items`)
2.  The initial selection for the item (`selected`)

Let's assume those properties are coming from our controller scope, through the `areas` object:

**app.js**
```javascript
angular.module("my-app")

.controller("AppController", ($scope) => {
  $scope.areas = {
    list: [
      "Floorplan",
      "Combinations",
      "Schedule",
      "Publish"
    ],
    current: "Floorplan"
  };
});

```

We can pass the properties into the directive using HTML attributes, so our application markup might look like this:

**index.html**

```markup
<ot-site>
  <ot-list items="{{ areas }}" selected="{{ areas.current }}"></ot-list>
</ot-site>
```

So our `ot-list` implementation will have to "catch" these properties from the controller and put them on our template scope. Remember that there are three ways to accomplish this: by setting a shared scope, a child scope, or an isolate scope.


### Shared scope
If we don't set the `scope` property, we can pass the data through by taking advantage of the `attrs` argument of the link function. We can transfer each item from `attrs` to the scope one-by-one:

**otList.js**
```javascript
angular.module("ot-components")

.directive("otList", function() {
  return {
    templateUrl: "ot-list.html"
    link: function (scope, elem, attrs) {
      scope.items = JSON.parse(attrs.items);
      scope.selected = attrs.selected;
              
      scope.selectItem = function(item) {
        scope.selected = item;
      };
    } 
  };
})
```

If we test that code, it will actually appear to work fine:

![One list](https://i.imgur.com/J3tKn17.png)

[Play with the code demo here](http://codepen.io/kara/pen/jPQeXw)

However, when you don't set the `scope` property at all, as we've done, the value is `false` by default.  This means that your directive creates **no** new scope of its own.  It shares the scope of whatever its outside context happens to be.  This means the scope of the directive is completely vulnerable to its outside environment - and vice versa. 

To drive that point home, let's see what happens when we add a second list to the application, drawing from a second data source in the controller, `apps`.

**app.js**
```javascript
angular.module("ot-components")

.controller("AppController", ($scope) => {
  $scope.areas = {...}; 
  $scope.apps = {
    list: [
      "Marketing",
      "Planning",
      "Reservations",
      "Settings"
    ],
    current: "Marketing"
  };
```

**index.html**
```markup
<ot-site>
  <ot-list 
   items="{{ areas.list }}" 
   selected="{{ areas.current }}">
  </ot-list>
  <ot-list
   items="{{ apps.list }}"
   selected="{{ apps.current }}">
  </ot-list>
</ot-site>
```

If we look at the output for the two lists... 

![Two lists (wrong)](https://i.imgur.com/xriTsxl.png)

...something is obviously off.  Check out [the code demo](http://codepen.io/kara/pen/MwzPZr) and click around.

We set up two different lists of data on the controller, one for each list instance, so we’d expect each list to show its own set of items.  But both of the lists are displaying the same data.   

And if we click on either of lists to select something, both of the lists show the new selection. They're glued together.  We would want each list to be selectable independently of other lists... so what’s going on?

As foreshadowed, shared scope is the culprit here.  As the list directives aren’t defining their own scopes, you’ll remember that both of their templates are bound to whatever outer scope they were placed in.  Since we have transcluded the lists into the site scaffold, they are sharing the `ot-site` transclusion scope.

![Shared scope diagram](https://i.imgur.com/zP2IyaD.png)

*Design credit: Simon Attley*

Since we can only have one `items` property and one `selected` property per scope, this means that the two list instances are sharing these properties. The first instance sets an `items` property and a `selected` property, then the second instance immediately overwrites them. That’s why the lists are the same, and the selections are coordinated.  We need to have a setup where the instances aren’t sharing variables and overwriting each other.

This setup also has another problem - even with one instance of `ot-list`.  What would happen if we dropped `ot-list` in an outer context that already had an `items` or `selected` variable.  In that case, the second instance of `ot-list` wouldn’t just overwrite the variables of the first instance. **It would also break whatever was using those variables in the broader application**.  You’re giving the directive the ability to pollute its outer environment and potentially create odd problems down the line.  Shared scope can be pretty risky. 

### Child scope

We can improve the situation by simply setting the scope property to `true`.   

**otList.js**
```javascript
angular.module("ot-components")

.directive("otList", function() {
  return {
    scope: true,
    templateUrl: "ot-list.html"
    link: function (scope, elem, attrs) {
      scope.items = JSON.parse(attrs.items);
      scope.selected = attrs.selected;
              
      scope.selectItem = function(item) {
        scope.selected = item;
      };
    } 
  };
})

```

When scope is `true`, each instance of the directive will create its own child scope in the outer scope. So each instance of `ot-list` here has its own copy of `items` and `selected`.  As sibling scopes, they won’t affect or overwrite each other’s variables. 


![Child scope diagram](https://i.imgur.com/VNSSYR3.png)

*Design credit: Simon Attley*


If we look at the output now that scope is `true`, we can see that our problem has been fixed.  Each list has its own set of data, and the selections move independently of one another.

![Two lists (correct)](https://i.imgur.com/IhMcmwV.png)

[See code demo here](http://codepen.io/kara/pen/VLVEOZ)

This is undoubtedly an improvement, but it too has its drawbacks.  Having created a child scope, the list is still part of the prototype chain.  While we’ve fixed any leaks from the list into its outer environment, what about leaks from its outer environment into the list?

I'll give an example.  Let’s say we wanted to add an optional header section to our list that would describe what the list contained.  If you added a `header` attribute to the directive and passed in header text, the list would display a header.  If there was no `header` property, the header section of the list wouldn't appear at all.  With a child scope, this setup would fail if a `header` property happened to exist anywhere above the directive in the prototype chain.

Why?  Let's say there was a `header` property on the controller scope for a different purpose, and we set up our `ot-list` directive without passing in a header.  We'd expect that no header section would appear on our list because we didn't pass one.  However, the `header` property from the controller would leak down to the directive scope through inheritance. The `header` property correctly wouldn't be found on the directive scope, but once that lookup failed, JavaScript would check the scope it inherits from - the controller - and would find and use that `header` variable. Thus, the directive would always show the text from the controller.

Any time you use a child scope, the child scope will always be vulnerable to pollution from up the prototype chain. So if the directive user happens to forget to add an attribute - or, like in this case, deliberately omits one - it might inherit an unrelated one from its environment.

### Isolate scope

For reusable components, we want complete assurance that there won’t be any leaks in either direction - it shouldn’t be able to affect its environment and its environment shouldn’t be able to affect it.  That way, we can be sure it will work in any context.  So what we need here is a scope that is outside this prototype chain, that won’t inherit anything directly from its environment - in other words, an isolate scope.

We can create an isolate scope as soon as we pass an object in to the scope property. It can simply be an empty object, as Angular is just checking its type.

**otList.js**
```javascript
angular.module("ot-components")

.directive("otList", function() {
  return {
    scope: {},
    templateUrl: "ot-list.html"
    link: function (scope, elem, attrs) {
      scope.items = JSON.parse(attrs.items);
      scope.selected = attrs.selected;
              
      scope.selectItem = function(item) {
        scope.selected = item;
      };
    } 
  };
})

```

What does this do to our scope hierarchy?  It takes each `ot-list` instance out of the prototype chain and completely isolates it.

![Isolate scope diagram](https://i.imgur.com/gTcS9Ku.png)

*Design credit: Simon Attley*

 It can’t inherit anything directly.  If we want our directives to have access to any variable, we will have to pass it in explicitly through the scope object. This creates a type of "whitelist", and has the added benefit of allowing us to remove this laborious movement of attributes one by one to the scope.


**otList.js**
```javascript
angular.module("ot-components")

.directive("otList", function() {
  return {
    scope: {
      items: "=items",
      selected: "=selected"
    },
    templateUrl: "ot-list.html"
    link: function (scope, elem, attrs) {
      scope.selectItem = function(item) {
        scope.selected = item;
      };
    } 
  };
})

```

If you use the scope object, you can use its shorthand instead.  On the left, you add the variables you want on the scope, and on the right, you place the attribute names that correspond to those variables. 

If the names will be the same, you shorten it further by omitting the names and keeping the binding strategy:

**otList.js**
```javascript
angular.module("ot-components")

.directive("otList", function() {
  return {
    scope: {
      items: "=",
      selected: "="
    },
    templateUrl: "ot-list.html"
    link: function (scope, elem, attrs) {
      scope.selectItem = function(item) {
        scope.selected = item;
      };
    } 
  };
})

```

Another advantage of this syntax is that it simplifies setting up two way binding.  Instead of manually setting up a `scope.$watch`, you can use the `=` binding strategy to accomplish the same thing.

This also allows us to remove the curly braces from our markup and pass our variables in directly for two-way binding:


**index.html**
```markup
<ot-site>
  <ot-list 
   items="areas.list" 
   selected="areas.current">
  </ot-list>
  <ot-list
   items="apps.list"
   selected="apps.current">
  </ot-list>
</ot-site>
```

If we run the code after all our improvements, the result will still work as expected:

![Isolate scope lists](https://i.imgur.com/rXr31BO.png)

[See code demo here](http://codepen.io/kara/pen/bdQQbW)

Isolate scope is pretty great in that it protects your directive from any outside influence. However, it’s worth noting that there are some specific cases where isolate scope might not be the right choice.  For instance, if you are creating an attribute directive designed to work with other directives on the same element, an isolate scope doesn't really make sense.  Only one isolate scope is allowed per element, so Angular would throw an error. 


## Angular 2 Directive scope

### Scope differences in Angular 2

With Angular 2 on the horizon, it's important to ensure that directives we write now are easily migratable.  To make our `ot-list` directive definition more portable, it would be wise to reduce our reliance on the `scope` and move that logic to the controller.

This is because in Angular 2, views will be automatically bound to the component class directly, which allows you to maintain any necessary state or functionality on the class itself.  As such, scope is superfluous as a concept and won't be a part of writing directives.

### Migrating directives to Angular 2

So how can we reduce our reliance on scope?

1.  First, we can move our models from the scope to the controller by setting the `bindToController` property to `true`.  This shifts our two-way bindings of `items` and `selected` from the scope to the controller itself (so from `$scope.items` to `ctrl.items`, etc). Now we are saving all state on our controller.

2.  Next, we can move our `selectItem` function to the controller by moving it into the `controller` function of the DDO and setting it to `this`.

3.  We can use the `controllerAs` property to give our template a reference to the controller (here we've set that reference as `ctrl`).  
  
4.  Lastly, in our template, we just have to update our references to `ctrl.items`, `ctrl.selected`, and `ctrl.selectItem`.
  

**otList.js**
```javascript
angular.module("ot-components")

.directive("otList", function() {
  return {
    scope: {
      items: "=",
      selected: "="
    },
    bindToController: true,
    controllerAs: "ctrl",
    templateUrl: "ot-list.html"
    controller: function() {
      this.selectItem = function(item) {
        this.selected = item;
      };
    } 
  };
})

```

**ot-list.html**
```markup
<ul>
    <li ng-repeat="item in ctrl.items" ng-bind="item" 
      ng-class="{'ot-selected': item === ctrl.selected}"
      ng-click="ctrl.selectItem(item)">
    </li>
</ul>
```

[See the code demo here](http://codepen.io/kara/pen/rVRMrX)

We can actually take this a step further.  Currently, we are setting up the controller as an anonymous function. To get as close as we can to Angular 2 component class syntax, we should pull it out into its own, named function, `ListController`.

**otList.js**
```javascript
angular.module("ot-components")

.directive("otList", function() {
  return {
    scope: {
      items: "=",
      selected: "="
    },
    bindToController: true,
    controllerAs: "ctrl",
    templateUrl: "ot-list.html"
    controller: ListController
  };
})

function ListController(){
  this.selectItem = function(item) {
    this.selected = item;
  }
}

```

[See updated code demo](http://codepen.io/kara/pen/EjMXGO)

This way, when migrating to Angular 2, you already have your component class set up and ready to go.

## Debugging tricks

### scope.$parent gotcha

One thing to watch out for when debugging scope problems is `$parent` property on the scope object.  At first glance, you may assume that this property points to the parent that the scope inherits from - and this is *sometimes* true.  

However, that's not a guarantee.  Though a transclusion scope inherits from the directive's outer context, its `parent` actually points to the directive scope.  It does **not** inherit from that scope.  The reference is set up this way to ensure that the transclusion scope is properly destroyed when the directive scope is destroyed.  

## Summary

We've explored the various types of scope that exist in directives and their strengths and weaknesses.  

Shared scope is risky for any directive with bindings, as it has the potential to overwrite properties and even break its outer environment. 

Child scope can be a happy medium between shared scope and isolate scope - inheriting from its parent, but not able to influence anything outside of itself. That said, there is potential for properties in its outer context to leak inside and disrupt its functionality.  

The only way to ensure that a directive's functionality is protected is to set up an isolate scope.  Given that data must be explicitly passed into the directive through the scope object, it is the safest choice.  However, it's not possible to use in all situations, given that only one isolate scope can be created per HTML element.

Lastly, we can't forget that transcluding directives have a second scope, a transclusion scope that will always be able to access models from the broader application.

I hope this overview was helpful.  For more information on Angular scopes, you may also want to check out <sup>[Documentation for Angular scope](https://docs.angularjs.org/guide/scope)</sup>.  

Happy scoping!

