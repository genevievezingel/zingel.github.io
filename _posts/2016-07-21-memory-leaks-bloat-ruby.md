
---
layout: post
title: "How to solve Memory Leaks or Bloats in Ruby while keeping your sanity."
description: "Memory issues are a nightmare for developers to solve. This doesn’t need to be the case with  Ruby’s Heap Dumps and `Heapy` gem. I will go into the weeds with a  real life example and how we solved our  memory issue."
date: 2016-07-21
read: "15 min read"
category: software
permalink: /blog/memory-leaks-bloat-ruby/
---

Usually having a memory leak is a terrible experience. It seems to follow Murphy’s law; occurring at the worst possible time and often late in the development cycle. You often don’t  know where to even begin all you know is that your application memory constantly increases overtime.

When a memory issue occurred on our site, I Initially upgraded ruby, as we were running a slightly older version, and the project’s gems with the  hope that someone else had discovered and fixed the issue. 

No luck there!

Unfortunately, I didn’t have the time to fix it and had to take a pragmatic option;  automating **reboot**  the  application when the memory usage goes  beyond a set threshold.  It does get you over the line, though it's a terrible solution as it kills the application’s performance while it's rebooting.

The following steps were used to solve the memory issue, and to remove the **reboot** temporary solution.

## How Ruby manages memory 
Since Ruby 2.1, there are new diagnostic methods to allow low level  Ruby introspection. But to use these tools, it first pays to have a good understanding of the `Heap`, `Kernel`, and `Garbage Collector` and how they relate to each other.

## Heap
The Heap stores the live Ruby objects. When the below code is executed, a number of objects are added to the Heap including the `class Foo`, `Foo object` along with a reference `foobar` to `Foo` object. 

~~~ ruby
class Foo

end

foobar = Foo.new
~~~
##  Kernal
The allocation of memory space, to store the Heap objects, is handled by the Kernal. If the programme continued to create objects, which are added to the Heap, there will come a time that the initial allocated space would be exhausted. It’s the Kernal’s responsibility to ensure that this doesn’t occur and allocates additional space as required.

However, once additional memory has been assigned to a Ruby process it doesn’t get re-assigned once it is no longer required<sup><a id="ffn1" href="#fn1" class="footnote">1</a></sup>. This was particularly important in what happened to our application. 

### The Garbage Collector (GB)
Take the example below, using Rail’s ActiveRecord, the following objects are added to the Heap:
1.  String object created from the method `name`. This object is referenced by `person_name` and contains the string value  `chris`
2.  Instance of `Person` class - populates the object with values from the   database’s  “person” table and the row corresponding to the record `id= 1`. 


~~~ ruby
person_name = Person.find(1).name
=> chris
~~~

What is important here is that once this code has run, these two objects are added to the Heap, but only one of the objects, `person_name` (1),  is reachable through the reference. 

The instance of `Person` (2) is unreachable as it wasn’t  assigned a  reference variable - while a new instance of `Person` can be created the original object  is unreachable;  its lost in the sea of objects.

The `Garbage Collector` is responsible for the removal of these  unreachable objects from the Heap, which reduces the  memory footprint, and improves the application’s performance. 

There are two strategies that Ruby uses to remove `unreachable objects`.

1.  Mark and Sweep strategy 
2.  Generational strategy

### Mark and Sweep Strategy 
The Garbage Collector traces the object hierarchy from root objects and marks each “visited” object to  **be kept**. Once finished, a “sweep” occurs which removes all unmarked  objects. In our example above, the ``person_name ` object would be marked but the `object instance of Person` would be a candidate for  removal when the next “sweep” occurs

### Generational Strategy
The implementation of the `mark and sweep strategy` is applied in two separate ways:
- Minor Garbage Collection 
- Major Garbage Collection

#### Minor Garbage Collection
The idea here is that newly created objects are more likely to be removed than objects that were created a while ago and have survived numerous `Mark and Sweeps`

> This is often seen in those nature shows, where a film crew follows a pack of animals over a year. The central theme often is the survival of the newly born animals. As a viewer, it can be a nail biting time as they struggle against the  odds of  surviving their first “critical year”. If they get past that first year, their survival rate dramatically improves.

Here the Garbage Collector only focuses on those “newly” created objects as a strategy to increase the likelihood of identifying  `unreachable`objects for garbage collection.

#### Major Garbage Collection
The Garbage Collector transverses the whole Heap for any objects that have become unreachable regardless of their age. Once a collection has occurred, any surviving objects that haven’t been collected before gets assigned to a new generation category. From this point onwards, all created objects will be kept separate for `Minor Garbage Collection`; they will become the next `Generation` when the next Major Garbage Collection  occurs. 

Note: The `Generation number` provides useful information on an object i.e. when and how many cycles ago it was created.

##  Stepping through our Investigation
Ubuntu utility, `htop` was used to examine which running processes were consuming the memory. Monitoring the memory usage, over a number of hours, the issue was with the `Sidekiq` processes as they increased in size from 30 MB to over 3000 MB.

> Sidekiq is a Ruby application that executes tasks in a  queue.  It works by adding any new task to the  queue, which is then executed  by `workers`. Once a task is finished, the `Worker` then  grabs the next task from the queue.  Typically, it is used when the user doesn’t need an instant response i.e. sending email, generating reports, or updating external APIs.

### Top view - Counting Objects on the Heap
 The number of objects on the heap was outputted using the `rake task` below. 

~~~ ruby
	namespace :memory do
	  desc "memory object count"
	  task :object_count => :environment do |t, args|
			Workers::Memory::ObjectCount.perform_async()
	  end
	end
~~~

It works by adding a task to Sidekiq that counts the number of objects on the Heap.

~~~ ruby
	class Workers::Memory::ObjectCount
	  include Sidekiq::Worker

	  def perform
		counts = 0
		ObjectSpace.each_object do |o|
		  counts += 1
		end

		Sidekiq.logger.info "Object count by Sidekiq: Total count #{counts}"
	  end
	end
~~~

 Initially, the number of objects on the heap climb over time, which certainly looks like a  memory leak. However, once the queue was emptied of tasks, the number of objects dramatically dropped. This is not consistent with a memory leak as the  number of objects would continue to trend upward. 

It suggests that the issue is **Memory Bloat**.  Where additional objects caused the Kernal to allocate further memory to a process. These additional objects would  eventually get garbage collected but the Ruby process doesn’t release the excess  memory back to the Kernal.

> **A quick review:** of what we know at this stage. There are a number  of objects being created, that are referenced, and survive Garbage Collection. These objects consume the available memory, and trigger the  Kernal to allocate additional memory. However this is not a memory leak as these objects eventually get collected by the Garbage Collector.

While the problem is Memory bloat, the steps to solve the issue is the same as a memory leak. 

###  A deeper dive into the Heap

#### Snapshot the Heap
Ruby provides a meta snapshot of the objects on the `Heap`, along with their `generation number` and where in the code they were created. To collect this information, the trace has to be activated by using the code below.

~~~ ruby
require 'objspace'
ObjectSpace.trace_object_allocations_start
~~~

The code ` ObjectSpace.dump_all` is then used to output objects on the `Heap`. In the below code, this is saved to a file for further analysis. 

~~~ ruby
  output = File.open("/app/dump/Heap#{self.class.counter}.json", "w")
  ObjectSpace.dump_all(output: output)
  output.close
~~~

To ensure that only `actively referenced` objects are captured, a sweep  is requested before the dump occurs using the code  `GC.start`. 

In the code below, a “Dump” occurs every 1000 Sidekiq tasks, using this code `self.class.counter % JOBS == 0`. 

The advantage of this is that we can compare the dumps over a period of time .

### Analysis of the Heap Dump
The most successful way that I found to analysing the Heap snapshots was to use the `Heapy` gem. This produces a couple of useful  summaries.

#### Summary of Number of Objects by Generation 
 This shows the number of objects on the Heap by their generation. If you are wondering why there is a  `nil` generation. These are the objects that were created prior to starting the `GC tracer`.

In the analysis below, the Heap  had a large number of initial objects, which is expected with the `rails boot-up process`. However, what wasn’t expected was a large number of objects created in generation 361.


~~~ ruby
 Analyzing Heap
==============
Generation: nil object count: 375147
Generation: 44 object count: 2717
Generation: 45 object count: 4832
Generation: 46 object count: 6490
Generation: 47 object count: 5363
miss a few
Generation: 361 object count: 108755
~~~

#### Summary of a Heap by file location
The `Heap` gem provides a summary breakdown for a generation by the location in the program where the object was created.  

This breakdown was applied to Generation 361 to find out what was the cause of all of these new objects. 

The summary below shows the  object’ count and their memory usage along with where in the code that they were created. The bulk of the objects where created in the `New Relic` gem directories.  After a quick internet search, I found that a number of users had experienced similar issues and they advised that `New Relic`  was not suitable for production usage.

> Though if you haven’t used  `New Relic` before give it a go. It is a fantastic diagnostic tool that gives insight into your application. I however will no be using it in production.

Once it was removed from the production server, the memory bloat disappeared. Awesome! 

~~~ ruby
Analyzing Heap (Generation: 361)
-------------------------------
allocated by memory (17588801) (in bytes)
==============================
13027511 /app/vendor/bundle/Ruby/2.3.0/gems/newrelic_rpm-3.15.2.317/lib/new_relic/agent/transaction/developer_mode_sample_buffer.rb:47
120632 /app/vendor/bundle/Ruby/2.3.0/gems/newrelic_rpm-3.15.2.317/lib/new_relic/agent/transaction/trace.rb:54
88400 /app/vendor/bundle/Ruby/2.3.0/gems/newrelic_rpm-3.15.2.317/lib/new_relic/agent/transaction_sampler.rb:192
~~~


#### Code for Memory Dump
The existing Sidekiq code was modified as follows 

**config/initializers/Sidekiq.rb**

~~~ ruby
	require "objspace"
	ObjectSpace.trace_object_allocations_start

	module Sidekiq
	  module Middleware
		module Server
		  class Profiler
			# Number of jobs to process before reporting
			JOBS = 1000

			class << self
			  mattr_accessor :counter
			  self.counter = 0

			  def synchronize(&block)
				@lock || = Mutex.new
				@lock.synchronize(&block)
			  end
			end

			def call(worker_instance, item, queue)
			  begin
				yield
			  ensure
				self.class.synchronize do
				  self.class.counter += 1

				  if self.class.counter % JOBS == 0
					Sidekiq.logger.info "reporting allocations after #{self.class.counter} jobs"
					GC.start
					output = File.open("/app/dump/Heap#{self.class.counter}.json", "w")
					ObjectSpace.dump_all(output: output)
					output.close
					Sidekiq.logger.info "Heap saved to Heap.json"
				  end
				end
			  end
			end
		  end
		end
	  end
	end

	Sidekiq.configure_server do |config|

	 config.server_middleware do |chain|
		chain.add Sidekiq::Middleware::Server::Profiler
	  end
	end
~~~

### References
- https://github.com/schneems/Heapy/blob/master/bin/Heapy
- https://blog.codeship.com/the-definitive-guide-to-Ruby-Heap-dumps-part-i/
- https://github.com/schneems/derailed\_benchmarks
- http://tenderlove.github.io/Heap-analyzer/


<ol id="footnotes">
<li id="fn1">This doesn’t apply to JRuby it allows reallocation of memory back to the Kernal for allocation <a href="#ffn1">&#x21A9;&#xFE0E;</a></li>
</ol>

