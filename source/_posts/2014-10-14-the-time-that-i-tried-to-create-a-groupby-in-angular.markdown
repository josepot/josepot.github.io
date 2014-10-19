---
layout: post
title: "Creating a 'groupBy' $filter in Angular"
date: 2014-10-14 01:00:56 -0400
comments: true
categories: AngularJs custom-filter ngRepeat
---
A few weeks ago I decided that I wanted to develop a 'groupBy' filter,
I knew that these kind of filters are tricky because they tend to generate
infinite loops in the `$diggest` cycle. However, I wanted to fully understand why
these kind of `$filter`s run into this problem and what was the best way
to overcome this issue.

In this post I will explain all the steps that I took for developing this `$filter`,
the problems that I encountered, all the things the I learned and the final implementation
of the `$filter`.

<!-- more -->

###The Goal

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

###First Naive Attempt

First I wrote this `$filter`:

{% highlight js %}
angular.module("josepot.filters", [])
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
})
{% endhighlight %}

When I tried it out I realized that there were errors in the console.

###The Problem

In order to trouble-shoot those errors I wrote these unit tests:

{% highlight js %}
{% raw %}
describe('Josepot\'s groupBy filter', function () {
  var $filterProvider, $compile, $scope, $filter, students;

  beforeEach(function(){
    module('josepot.filters');
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

    students = [
        {ID: 1,  name: 'Josep',  class: 'A'},
        {ID: 2,  name: 'Carles', class: 'B'},
        {ID: 3,  name: 'Xavi',   class: 'A'},
        {ID: 4,  name: 'Pere',   class: 'B'},
        {ID: 5,  name: 'Adrià',  class: 'C'}
    ];
  });

  it('should group students by class', function(){
    var grouppedStudents = $filter('groupBy')(students, 'class');
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
    $scope.students = students;
    var elem = angular.element("<p>{{students | groupBy: 'class'}}</p>");
    $compile(elem)($scope);
    expect($scope.$digest.bind($scope)).not.toThrow();
  });

  it('should work in a view combined with ngRepeat', function() {
    $scope.students = students;
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
  });
});
{% endraw %}
{% endhighlight %}

[These tests](http://plnkr.co/edit/M2mXjsWrd16uAad2dOXS?p=preview) gave me very interesting feedback:

 - This `$filter` worked well when used in JavaScript
 - It also worked well when used inside a trivial HTML template
 - It triggered an ["*Inifite $diggest Loop Error*" (`infdig`)][1] when used inside a [`ngRepeat` directive][2]

###The `$diggest` cycle always rings (at least) twice

The `$diggest` cycle is the stage in which Angular ensures the changes of the model have settled,
so that it can render the view with the updated changes of the model. In order to do that,
Angular starts a loop, in which each iteration will evaluate all the template expressions
of the View as well as its `$watcher`s and it will keep a copy of the result.
If in the current iteration the result is the same as the result of the previous one,
then Angular will leave there, otherwise it will try again. If after 10 attempts things
haven't settled, then Angular will exit with an error: The ["*Inifite $diggest Loop Error*" (`infdig`)][1].

It's still not obvious why this `$filter` is causing that error, after all the `$filter`
passed the second unit test successfully:

{% highlight js %}{% raw %}
  it('should work in a view', function() {
    $scope.students = students;
    var elem = angular.element("<p>{{students | groupBy: 'class'}}</p>");
    $compile(elem)($scope);
    expect($scope.$digest.bind($scope)).not.toThrow();
  });
 {% endraw %}{% endhighlight %}

Which means that the `$diggest` cycle haven't had any issues evaluating an Angular
Expression containing this `$filter`.

Then, why is the `$filter` failing when used insed an `ngRepeat` directive?  

Because the `ngRepeat` directive adds a `$watch`er into its container's `$scope`
for the collection that it's being iterated.
If [we have a look at the code of the `ngRepeat` directive][3] we'll find a
line like this one:

{% highlight js %}{% raw %}
    $scope.$watchCollection(rhs,
    	function ngRepeatAction(collection) {...
 {% endraw %}{% endhighlight %}

Which means that in our case, the `ngRepeat` directive is doing this:

{% highlight js %}{% raw %}
    $scope.$watchCollection("students | groupBy: 'class'",
    	function ngRepeatAction(collection) {...
{% endraw %}{% endhighlight %}

And this is what is causing the error, because our `$filter` is returning a new `Object`
containing new `Arrays` every time that it runs, that's why the `$diggest` cycle gets
into an infinite loop.

Let's write a new test to make sure that that's the source of problem:

{% highlight js %}{% raw %}
  it('should be able to $watchCollection for an expression using the groupBy',
  function() {
    $scope.students = students;
    $scope.$watchCollection("students|groupBy:'class'", function(collection){});
    expect($scope.$digest.bind($scope)).not.toThrow();
  });
{% endraw %}{% endhighlight %}

And if we run this test we will confirm that this test is also throwing the
["*Inifite $diggest Loop Error*" (`infdig`)][1].

###The Solution(s)

Before I implemented this solution I actually tried 2 different techniques for "stabilizing" the `$filter`:

* The first one (pretty bad in my opinion) is the one used by [Ariel Mashraki (a8m)][4] and that you can find [here][5].
* The second one (much much better) is the one that [Johnny Hauser (m59peacemaker)][6] uses for stabilizing his "unstable" `$filters`, which is this [Filter Stabilizer][7] that relies on Memoization.

At the end of this post there are 2 appendixes where I discuss the different reasons that
I had for not using any of these techniques for this particular `$filter`.

Without further ado, lets have a look at the final version of my  `$groupBy` `$filter`:

{% highlight js %}{% raw %}
angular.module("josepot.groupBy", [])
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

        for(var k in result)
          result[k].splice(0,result[k].length);

        for (var i=0; i<data.length; i++) {
            if (!result[data[i][key]])
                result[data[i][key]]=[];
            result[data[i][key]].push(data[i]);
        }

        var keys = Object.keys(result);
        for(var i=0; i<keys.length; i++){
          if(result[keys[i]].length===0)
            delete result[keys[i]];
        }
        return result;
    };
})
{% endraw %}{% endhighlight %}

I want this `$filter` for using it inside the `ngRepeat` directive and the `ngRepeat`
directive creates a new `$scope`, also I can't think of a single case where I would use this filter more than once inside an `ngRepeat`, therefore I decided that it was a safe assumption to use the `$scope.id` as an identifier of the results that I want to return. And if the  `$filter` is not used from a template and there is no `$scope` then I don't need to worry about the results being persistent and I'm safe creating a new `Object`.


  [1]: https://docs.angularjs.org/error/$rootScope/infdig
  [2]: https://docs.angularjs.org/api/ng/directive/ngRepeat
  [3]: https://github.com/angular/angular.js/blob/master/src/ng/directive/ngRepeat.js
  [4]: https://github.com/a8m
  [5]: https://github.com/a8m/angular-filter/blob/master/src/_filter/collection/group-by.js
  [6]: https://github.com/m59peacemaker
  [7]: https://github.com/m59peacemaker/angular-pmkr-components/tree/master/src/services/filterStabilize
