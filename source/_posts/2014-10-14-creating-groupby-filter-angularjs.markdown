---
layout: post
title: "Creating a 'groupBy' $filter in Angular"
date: 2014-10-14 01:00:56 -0400
comments: true
categories: AngularJs custom-filter ngRepeat unit-testing jasmine
---
A few weeks ago I decided that I wanted to develop a 'groupBy' `$filter`.
I knew that these kind of filters are tricky because they tend to generate
infinite loops in the `$diggest` cycle. However, I wanted to fully understand why
these kinds of `$filter`s run into this problem. I also wanted
to figure out the best solution to overcome this issue.

In this post I will explain all the steps that I took to develop this `$filter`,
the problems that I encountered, what I learned, along with the final implementation
of the `$filter`.

<!-- more -->

##The Goal

I wanted my custom 'groupBy' `$filter` to work like this:

* Given an `Array` of `Object`s like this one:

{% highlight js %}
   $scope.students = [
        {ID: 1,  name: 'Josep',  class: 'A'},
        {ID: 2,  name: 'Carles', class: 'B'},
        {ID: 3,  name: 'Xavi',   class: 'A'},
        {ID: 4,  name: 'Pere',   class: 'B'},
        {ID: 5,  name: 'Adrià',  class: 'C'}
   ];
{% endhighlight %}


* I wanted to be able to do this:

{% highlight html %}
{% raw %}
<ul>
    <li ng-repeat="(class, classStudents) in (students | groupBy: 'class')">
        {{class}}
        <ul>
            <li ng-repeat="student in classStudents">
                {{student.name}}
            </li>
        </ul>
    </li>
</ul>
{% endraw %}
{% endhighlight %}

##First Naive Attempt

First I wrote this `$filter`:

{% highlight js %}
angular.module("sbrpr.filters", [])
.filter('groupBy', function () {
    return function (data, key) {
        if (!(data && key)) return;
        var result = {};
        for (var i=0;i<data.length;i++) {
            if (!result[data[i][key]])
                result[data[i][key]]=[];
            result[data[i][key]].push(data[i])
        }
        return result;
    };
});
{% endhighlight %}

When [I tried it out](http://jsfiddle.net/xf7w9h12/1/) I realized that there were errors in the console.

##The Problem

In order to troubleshoot those errors I wrote these unit tests:

{% highlight js %}
{% raw %}
describe('groupBy filter', function () {
  var $filterProvider, $compile, $scope, $filter;

  beforeEach(function(){
    module('sbrpr.filters');
    inject(function (_$filter_, _$rootScope_, _$compile_) {
      $filter = _$filter_;
      $compile = _$compile_;
      $scope = _$rootScope_;
    });

   jasmine.addMatchers({
        toEqualData: function(angular) {
          return{
            compare: function(actual, expected){
              if (expected === undefined)
                expected = '';
              var result = {};
              result.pass = angular.equals(actual, expected);
              return result;
            }
          }
        }
    });

    $scope.students = [
        {ID: 1,  name: 'Josep',  class: 'A'},
        {ID: 2,  name: 'Carles', class: 'B'},
        {ID: 3,  name: 'Xavi',   class: 'A'},
        {ID: 4,  name: 'Pere',   class: 'B'},
        {ID: 5,  name: 'Adrià',  class: 'C'}
    ];
  });

  it('should group students by class', function(){
    var grouppedStudents = $filter('groupBy')($scope.students, 'class');
    var expectedResult = {
      'A': [
        {ID: 1,  name: 'Josep',  class: 'A'},
        {ID: 3,  name: 'Xavi',   class: 'A'}
        ],
      'B': [
        {ID: 2,  name: 'Carles', class: 'B'},
        {ID: 4,  name: 'Pere',   class: 'B'}
        ],
      'C': [
        {ID: 5,  name: 'Adrià',  class: 'C'}
        ],
    };
    expect(grouppedStudents).toEqualData(expectedResult);
  });

  it('should work in a view', function() {
    var elem = angular.element("<p>{{students | groupBy: 'class'}}</p>");
    $compile(elem)($scope);
    expect($scope.$digest.bind($scope)).not.toThrow();
  });

  it('should work in a view combined with ngRepeat', function() {
    var elem = angular.element(
      "<ul>" +
       "<li ng-repeat=\"(class, classStudents) in (students | groupBy: 'class')\">" +
          "{{class}}" +
          "<ul>" +
            "<li ng-repeat=\"student in classStudents\">" +
              "{{student.name}}" +
            "</li>" +
          "</ul>" +
        "</li>" +
      "</ul>");
    $compile(elem)($scope);
    expect($scope.$digest.bind($scope)).not.toThrow();
    expect(elem.children().length).toBe(3);
    expect(elem.children().children().children().length).toBe(5);
  });
});
{% endraw %}
{% endhighlight %}

[The results of these tests](http://plnkr.co/edit/22O4xbU0wweLCJws7qZC?p=preview) provided interesting feedback:

 - The `$filter` worked well when used in JavaScript
 - It also worked well when used inside a trivial HTML template
 - It triggered an ["*Inifite $diggest Loop Error*" (`infdig`)][1] when used inside a [`ngRepeat` directive][2]

###The `$diggest` cycle always rings (at least) twice

The `$diggest` cycle is the stage in which Angular ensures the changes of the model have settled,
so that it can render the view with the updated changes. In order to do that,
Angular starts a loop in which each iteration evaluates all the template expressions
of the view, as well as the `$watcher` functions of the `$scope`.
If in the current iteration the result is the same as the previous one,
then Angular will exit the loop. Otherwise, it will try again.
If after 10 attempts things haven't settled, Angular will exit
with an error: The ["*Inifite $diggest Loop Error*" (`infdig`)][1].

Despite this, it's still not obvious why the `$filter` is causing that error.
After all, the `$filter` passed the second unit test successfully:

{% highlight js %}{% raw %}
  it('should work in a view', function() {
    $scope.students = students;
    var elem = angular.element("<p>{{students | groupBy: 'class'}}</p>");
    $compile(elem)($scope);
    expect($scope.$digest.bind($scope)).not.toThrow();
  });
 {% endraw %}{% endhighlight %}

This means that the `$diggest` cycle hasn't had any issues evaluating an Angular
Expression containing this `$filter`.

So, why is the `$filter` failing inside a `ngRepeat` directive?  

It's because the `ngRepeat` directive adds a `$watch`er into its container's `$scope`
for the collection that's being iterated.
If [we have a look at the code of the `ngRepeat` directive][3] we'll find a
line like this one:

{% highlight js %}{% raw %}
    $scope.$watchCollection(rhs, function ngRepeatAction(collection) {...
 {% endraw %}{% endhighlight %}

Which means that in our case, the `ngRepeat` directive is doing this, which is causing the error:

{% highlight js %}{% raw %}
    $scope.$watchCollection("students | groupBy: 'class'", function ngRepeatAction(collection) {...
{% endraw %}{% endhighlight %}

Since the `$filter` is returning a new `Object`
containing new `Arrays` every time it runs, this causes the `$diggest` cycle to get
into an infinite loop for the `$watchCollection` function.

So, I wrote a new test to make sure that was the source of problem:

{% highlight js %}{% raw %}
  it('should be able to $watchCollection for an expression using the groupBy',
  function() {
    $scope.students = students;
    $scope.$watchCollection("students|groupBy:'class'", function(collection){});
    expect($scope.$digest.bind($scope)).not.toThrow();
  });
{% endraw %}{% endhighlight %}

[After running it](http://plnkr.co/edit/sRUSFy0FpmmSdC5NFm1f?p=preview), I confirmed this test was also throwing the
["*Inifite $diggest Loop Error*" (`infdig`)][1].

##The Solution(s)

Before I implemented my solution I tried 2 different techniques for "stabilizing" the `$filter`:

* The first one (pretty bad in my opinion) is used by [Ariel Mashraki (a8m)][4] which you can find [here][5].
* The second one (much better) is the one that [Johnny Hauser (m59peacemaker)][6] uses for stabilizing
his "unstable" `$filters`, which is this [Filter Stabilizer][7] that relies on Memoization.

Lets have a look at these 2 options.

### Ariel Mashraki's Solution

His solution is equivalent to doing this:

{% highlight js %}{% raw %}
.filter('groupBy', function ($timeout) {
    return function (data, key) {
        if (!key) return data;
        var outputPropertyName = '__groupBy__' + key;
        if(!data[outputPropertyName]){
            var result = {};  
            for (var i=0;i<data.length;i++) {
                if (!result[data[i][key]])
                    result[data[i][key]]=[];
                result[data[i][key]].push(data[i]);
            }
            Object.defineProperty(data, outputPropertyName, {enumerable:false, configurable:true, writable: false, value:result});
            $timeout(function(){delete data[outputPropertyName];},0,false);
        }
        return data[outputPropertyName];
    };
})
{% endraw %}{% endhighlight %}

Basically what it's doing is creating a non-enumerable property in the original `Array`
that is being filtered with the result of the 'groupBy'. In this way, when the `$diggest`
cycle triggers the `$filter`, the `$filter` will first check if the property has already been set.
If it hasn't, it will do the 'groupBy' and will save the result in the non-enumerable property.
If the property has already been set, it will return the cached value. Also, notice that
there is a `$timeout` that deletes that property after the `$diggest` cycle has finished.

I must admit that this is a clever way to try to trick the `$diggest` cycle and that this `$filter`
would pass the unit tests. However, this `$filter` has an issue: it can't be
combined with other `$filter`s. For example, [it wouldn't pass this test](http://plnkr.co/edit/SSw0mKePtP8NvYNO00uO?p=preview):

{% highlight js %}{% raw %}
  it('should work inside an ngRepeat in combination with other filters', function() {
    var elem = angular.element(
      "<ul>" +
       "<li ng-repeat=\"(class, classStudents) in (students|filter:'e' | groupBy:'class')\">" +
          "{{class}}" +
          "<ul>" +
            "<li ng-repeat=\"student in classStudents\">" +
              "{{student.name}}" +
            "</li>" +
          "</ul>" +
        "</li>" +
      "</ul>");
    $compile(elem)($scope);
    expect($scope.$digest.bind($scope)).not.toThrow();
    expect(elem.children().length).toBe(2);
    expect(elem.children().children().children().length).toBe(3);
  });
{% endraw %}{% endhighlight %}

### Johnny Hauser's "Filter Stabilizer" Solution

I must say that this [Filter Stabilizer][7] is **awesome** as a generic solution
for stabilizing `$filter`s, and that if it was used for the original filter, like this,
[it would pass all the tests](http://plnkr.co/edit/uAUE9g9STrMpsWc43vYO?p=preview):

{% highlight js %}{% raw %}
angular.module("pmkr.filters", [])
.filter('groupBy', ['pmkr.filterStabilize', function(stabilize){
    return stabilize( function (data, key) {
        if (!(data && key)) return;
        var result = {};
        for (var i=0;i<data.length;i++) {
            if (!result[data[i][key]])
                result[data[i][key]]=[];
            result[data[i][key]].push(data[i])
        }
        return result;
    });
}])
.factory('pmkr.filterStabilize', [
  'pmkr.memoize',
  function(memoize) {
    function service(fn) {
      function filter() {
        var args = [].slice.call(arguments);
        // always pass a copy of the args so that the original input can't be modified
        args = angular.copy(args);
        // return the `fn` return value or input reference (makes `fn` return optional)
        var filtered = fn.apply(this, args) || args[0];
        return filtered;
      }
      var memoized = memoize(filter);
      return memoized;
    }
    return service;
  }
])
.factory('pmkr.memoize', [
  function() {
    function service() {
      return memoizeFactory.apply(this, arguments);
    }
    function memoizeFactory(fn) {
      var cache = {};
      function memoized() {
        var args = [].slice.call(arguments);
        var key = JSON.stringify(args);
        var fromCache = cache[key];
        if (fromCache) {
          return fromCache;
        }
        cache[key] = fn.apply(this, arguments);
        return cache[key];
      }
      return memoized;
    }
    return service;
  }
]);
{% endraw %}{% endhighlight %}

There is only one small inconvenience though: **memory**. If this technique is used
with a large `Array` of objects and many changes are made to it
(i.e. add/remove items and/or filter the Array before grouping it),
then, every time that the Array changes, the `memoizeFactory` will serialize it.
And it will then store that as the `key` of the cache used for 'memoizing' the result of
the 'groupBy' function.

I must admit in most cases that shouldn't be an issue, but
still, I don't want to have to worry about possible (although unlikely) memory leaks.

### My Solution

This solution relies on the fact that the `ngRepeat` directive creates a new `$scope`,
the `$id` which can be used for identifying the cached result of the `$filter`. Also,
it's possible to listen for the `$destroy` event of that `$scope` to remove
its cached result in order to avoid memory leaks.

{% highlight js %}{% raw %}
angular.module("sbrpr.filters", [])
.filter('groupBy', function () {
  var results={};
    return function (data, key) {
        if (!(data && key)) return;
        var result;
        if(!this.$id){
            result={};
        }else{
            var scopeId = this.$id;
            if(!results[scopeId]){
                results[scopeId]={};
                this.$on("$destroy", function() {
                    delete results[scopeId];
                });
            }
            result = results[scopeId];
        }

        for(var groupKey in result)
          result[groupKey].splice(0,result[groupKey].length);

        for (var i=0; i<data.length; i++) {
            if (!result[data[i][key]])
                result[data[i][key]]=[];
            result[data[i][key]].push(data[i]);
        }

        var keys = Object.keys(result);
        for(var k=0; k<keys.length; k++){
          if(result[keys[k]].length===0)
            delete result[keys[k]];
        }
        return result;
    };
});
{% endraw %}{% endhighlight %}

It [passes all the tests][8], and the only situation where this would fail would be if the `$filter` was used more than once
inside the same `$scope`, but I can't think of a single case where this would actually be a problem.

  [1]: https://docs.angularjs.org/error/$rootScope/infdig
  [2]: https://docs.angularjs.org/api/ng/directive/ngRepeat
  [3]: https://github.com/angular/angular.js/blob/master/src/ng/directive/ngRepeat.js
  [4]: https://github.com/a8m
  [5]: https://github.com/a8m/angular-filter/blob/master/src/_filter/collection/group-by.js
  [6]: https://github.com/m59peacemaker
  [7]: https://github.com/m59peacemaker/angular-pmkr-components/tree/master/src/services/filterStabilize
  [8]: http://plnkr.co/edit/Trr3dIEplKxODk4WpqTr?p=preview
