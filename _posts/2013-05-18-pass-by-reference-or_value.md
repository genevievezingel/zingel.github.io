---
layout: post
title: "How do method variables work in Ruby"
description: "The inspiration of this blog post has been a recent question on Stack Overflow about whether Ruby passes the method variables by value or by reference."
date: 2013-05-18
read: "11 min read"
category: software
permalink: /blog/pass-by-reference-or_value/
---

The inspiration of this blog post has been a recent question on Stack Overflow about whether Ruby passes the method variables `by value` or `by reference`. The short answer, is that it is passed `by reference`, or perhaps more accurate by <a href ="http://en.wikipedia.org/wiki/Evaluation_strategy#Call_by_sharing">"called by sharing"</a>

Let's take a look at the example (slightly modified) that was causing the confusion. In the below example, it looks like the value is being passed, as the `address` variable hasn't been changed by the `down_case` method.

~~~ ruby
 address = "123 ABC strEet"

 def down_case(address)
	address= address.downcase
 end

 puts address 
 =>123 ABC strEet

 down_case(address) 
=> 123 abc street

 puts address 
=>"123 ABC strEet"
~~~

That is incorrect. There are a couple of things going on, so let me take you on a journey (stay with me)

  -  how object reference works
  -  how scope of method variables works
  -  how passing the object's reference works in methods

## How Object References Work
An object can have multiple references to it; any of these references can be used to send messages to that  object to change its state.

In the example below. I created an object with the value "I am an object" that the variable `a` references. I later make `b` point to the same object.

Both of these references point to the same object. If you change `a's` objects it also changes `b's ` object as they reference the same object.

~~~ ruby
 a="I am an object"
 b=a
 puts b #=> "I am an object"
 puts a.object_id #=> 70307771825720
 puts b.object_id #=> 70307771825720
~~~



## How Scope of Method variables works

The method variable is local to the method even if it shares the same name as variable outside the method. 

In my example above, the `address` variable  exist both in the method scope and the outer scope.

The advantage here is that the same name in a method variable will not  modify the outer scope variable by “accidentally”  re-assigning it.  However, in the original example, the user wanted to change the outer scope variable, but instead change the inner scope `address` reference to point to the new "123 abc street" String Object, without impacting  the outer scoped `address` reference  which remained pointing to the "123 ABC strEet" String Object.

## How Passing the Object's reference works in Methods

To get the users desired result the method `downcase!`could have been used to modify the outer object.

This works by modify the underlying object i.e. modifies the object to "123 abc street” from "123 ABC strEet”. Which is different from `downcase` method as it creates a new object with the value "123 abc street" which is assigned the `address` reference, with the  inner and outer scope `address` pointing  to different objects with different values.

 This passing the reference of object, but not the actual reference is know as <a href ="http://en.wikipedia.org/wiki/Evaluation_strategy#Call_by_sharing"> called by sharing </a>

~~~ ruby
 address = "123 ABC strEet"

 def down_case(address)
	address.downcase!
 end

 puts address 
  =>"123 ABC strEet"

 down_case(address) _
   =>"123 abc street"

 puts address 
=>"123 abc street"
~~~


##How else could this have been done
You could also use Procs or Lambdas to get a similar results.  A  Proc has access to the  scope that existed when it was created.

~~~ ruby
 address = "123 ABC strEet"

 down_case = Proc.new do |a|
	 address =a.downcase
 end

 down_case.call(address)
 puts address
~~~

The `downcase` method call creates a new string object with the value "123 abc street" but since the `address` reference existed before the creation of the Proc it is “in scope” so  can be re-assigned to the  new String Object  "123 abc street" i.e. this changes the `address` reference outside the Proc method.

## Summary
We pass in references rather than values to method, but the method variable is local to the function  ( <a href ="http://en.wikipedia.org/wiki/Evaluation_strategy#Call_by_sharing">see  called by sharing for more details</a>) even if they share the same name

If you want to share scope consider using a Proc or Lambda

##References
- <a href="http://ruby-doc.org/core-2.0/Proc.html"> Procs </a>
- <a href ="http://en.wikipedia.org/wiki/Evaluation_strategy#Call_by_sharing">called by sharing</a>
