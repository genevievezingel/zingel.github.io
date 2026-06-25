---
layout: post
title: "Don't Use Class variables.... Really"
description: "The reason why you should use Ruby's <strong>class instance variables</strong> instead of class variables"
date: 2015-12-12
read: "10 min read"
category: software
permalink: /blog/class-variable/
---

I recently read on **stack overflow** that you shouldn't use `class variables` but instead use  ` class instance variables`. I wasn’t satisfied in the reason given, and hence the short blog piece below.

##The Confusion

The problem with ` class variables`, in Ruby, is that the scope is not what you expect. This is especially true if you come from other languages such as Java.

In Java `class variables` belongs to the **class**, which is different to Ruby  where they belong to the **class hierarchy**. 

This was the source of my confusion it would have been better if they were called  `class hierarchy variable`.

## Here is how class variables bites:


~~~ ruby
#------------------------------------------------------#
# The class variable starts as "happy" but ends        #
#  very "sad"                                          #
#------------------------------------------------------#

class MyParent
  def MyParent.get_happy_value
	@@happy_variable
  end
  @@happy_variable="this make's me happy"
end

MyParent.get_happy_variable #=> "this make's me happy"

class MyChild <MyParent
  @@happy_variable="this make's me sad"
end

MyParent.get_happy_variable #=> "this make's me sad"
~~~
Let's take a look at what is going on

<img src="{{ '/assets/images/blog/really_do_not_use_class_variables/incorrect_example.jpg' | relative_url }}" alt="VSCode" itemprop="image"  class="u-photo">


The `class variables` `@@happy_variable ` does not belong to either the class or object but rather the class hierarchy, so it can be changed by a child class with dire results. The `@@happy_variable` refers to the same variable across both classes. This wouldn't be the case, if you wrote the same class method in another language like Java.

## So how do I create a class variable (like in Java)

To get the same behaviour use ` class instance variables.` It is essentially just an instance variable but at the class level ie it belongs to the class .


~~~ ruby
#------------------------------------------------------#
# The class instance variable starts as "happy" stays  #
#  very "happy"                                        #
#------------------------------------------------------#

class MyParent
  @happy_variable="this make's me happy"     #lives in the class not the instance
  def get_happy_value
  self.class.instance_eval{@happy_variable}  #changes self from the instance to the class
  end
end

MyParent.new().get_happy_value  #=> "this make's me happy"

class MyChild <MyParent
  @happy_variable="this make's me sad"
end

MyParent.new().get_happy_value  #=> "this make's me happy"

~~~

Since the ` instance class variables ` belongs to the class each class can have a different value for `@happy_variable` This is different to ` class variables ` where the variable belongs to the class hierarchy. Here any changes to `@@happy_variable` will be for all of the classes in the hierarchy.
