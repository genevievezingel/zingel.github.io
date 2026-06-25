---
layout: post
title: "Intro into Rack"
description: "If you are aware of Rack, seeing it crops up in the  <code>stack trace</code>, but you just haven’t gotten around to “understanding”  it then this  is the blog post for you.</p> <p>I go through the Rack standard, before diving into how it  intuitively works.  With the bold promise that you will be a better programmer if you understand Rack as it is the backbone for  a number of popular frameworks such as Rails and Sinatra."
date: 2017-02-05
read: "8 min read"
category: software
permalink: /blog/intro_into_rack/
---

### A look into how Rack works 
This weeks  writing topic is  to understand how Rack works.  Rack provides a  standardised interface with the web server  and is used by a  number of  web frameworks including Sinatra and Rails. 

I had previously been aware of Rack, and how it was the back bone of Rails, but hadn’t  put in the hard yards to understand it. This post outlines how it intuitively works and hopefully removes some of the mystery surrounding Rack. 

### What is Rack
Imagine that your web application  is made up of different layers. These layers will have different responsibilities from authentication, to adding content. A web request  then goes  through these  layers  and the resulting  response is  returned. 

Having a standardised set of rules that each layer must  comply  with allows us to  develop “new” layers that will “just work” with  the existing  layers as long as it complies with this standard. This standardised rules  is know as Rack.  As it turns out, the standard is remarkably simple to  implement.

#### Rack Specification

The Rack specification requires that the middleware must have  a instance method called `call` and unless it is the last middleware in the queue must stores a reference to the next middleware. The  ‘call’ instance method  takes an environment hash as a parameter, and returns an Array with three elements. 

*These  elements are:*

- The HTTP response code
- A Hash of headers
-  and a response body

The layers are linked together via the `call` method. The middleware of the first layer will execute the `call` method of the next layer using the  stored reference.  While that `call` method is being executed it also executes the `call ` method of the next layer, and so on.

(See  [github.io](https://rack.github.io/) for more details)

### Seeing is believing 
I found walking through an example helped me understand how it all fits together. The code below has been modified to aid understanding.  If you want to see an  unadulterated version  then head to  the file  `/lib/rack/builder.rb` of the   `rack` gem.

~~~ ruby
class Header
  def initialize(app) @app = app; end
  def call(env)
	status, headers, response = @app.call(env)
	[status, headers.merge({"X-Authorization" => "secret"}), response]
  end
end

class Body
  def initialize(app) @app = app; end
  def call(env)
	status, headers, response = @app.call(env)
	[status, headers, ["#{response} from Body"]]
  end
end

class Builder
  def initialize(default_app = nil, &block)
	@use, @map, @run, @warmup = [], nil, default_app, nil
	instance_eval(&block) if block_given?
  end

  def use(middleware, *args, &block)
	@use << proc { |app| middleware.new(app, *args, &block) }
  end

  def to_app
	app =  @run
	app = @use.reverse.inject(app) { |a,e| e[a] }
	app
  end

  def call(env)
	to_app.call(env)
  end

  def run(app)
	@run = app
  end
end

builder_app = Builder.new {
  use Header
  use Body

  run  lambda { |env| [200, {}, ""] }
}

builder_app.call({})  #environmental variables from server

=> [200, {"X-Authorization"=>"secret"}, [" from Body"]]

~~~

### Specification  Middleware 
Let’s take the above code apart by looking at how the builder object is instantiated (created)

~~~ ruby
builder_app = Builder.new {
  use Header
  use Body

  run  lambda { |env| [200, {}, ""] }
}
~~~ 

The `Builder.new` receives a block that specifies which middleware is used in the  rack stack and in the order that they are  applied . In the case above, the `#use` method adds the Header and Body middleware. These  middleware  are required to have a  `call` instance method, and to store a reference to the next  middleware in the queue.

The last middleware in the queue is a special case since it  doesn’t need to store a reference to the next middleware. It gets added to the rack stack using the   above `run` method. Its only requirements is that it has a `call` method.  If you are wondering, this can be a lambda object, as seen in our example, since that gets  executed with  a `call` method.


###  How rack gets used?
We have finally arrived at how Rack gets used in our application. A server request comes in from our website. The  server then requests a response from our application by executing the following code. This is the entry point into our application, or ground zero.

`builder_app.call (environmental hash)` 

### What happens with builder_app.call
The above code returns the application response (status, body, headers) to the server and also creates the initial rack stack 

  `builder_app.call (environmental hash) `

  <img src="{{ '/assets/images/blog/rack/builder_create_rack_middleware.png' | relative_url }}" alt="Middleware" itemprop="image"  class="u-photo">


In our example, the `builder_app` references an  instance of  the `Builder`. This specifies the middleware  stack  while it is instantiated (see  above Specification  Middleware)

Thea above method runs the code  `to_app#call(env)`does two things *(step 1)*.:

-  The first part `to_app`method instantiates the rack stack (using the middleware specified in  `Builder.new` ) . 
- The  second part `#call` executes the call method on the instantiated rack stack 

Lastly, the `call method` starts off the  chain reaction by executing the  first middleware’s call method  in the rack stack . This in turn, executes the `call` method of the  next middleware in the rack stack, and so on.

### Instantiating the  middleware stack 
In our example, there are three middleware layers: `Header`, `Body` and `Lamba` object.  The `@use.reverse.inject` step ineffectively combines the code as `Header.new(Body.new(app)))` using ruby’s *inject* method.

~~~ ruby
  def to_app
	app =  @run
	app = @use.reverse.inject(app) { |a,e| e[a] }
	app
  end
~~~ 

The last middleware (lambda object) has no need to store the next middleware in the  queue. It only is required to have a `call` method that returns an array of *status*, *headers* and *body*.  

If it is not last,  the middleware stores a reference to the middleware next in queue.    In our example, the   `header` middleware  stores a reference  to the  `body` middleware *(step3)* and the `body` middleware stores a reference to the lambda *(step 4)*

<img src="{{ '/assets/images/blog/rack/builder_reverse_inject.png' | relative_url }}" alt="VSCode" itemprop="image"  class="u-photo">


What is important here is that the middleware are linked together by storing a reference to the middleware next in queue.

### The call method
After the code  `builder_app.call (environmental hash)`  has created the rack stack it  then executes the `call` method against it.  The `header object#call` is initially executed, but within that method it  kicks off the `call` method of the next  middleware using its  stored  reference `@app` **(step 7)**

<img src="{{ '/assets/images/blog/rack/through_the_rack_middleware.png' | relative_url }}" alt="VSCode" itemprop="image"  class="u-photo">


Within that method, the` body object’s` `call` method  is executed (step 8) and finally within this `call` method the  last  middleware’s `call` method is executed **(step 9)**.  

Phew ! The `call`method, for the last middleware object, completes  and the process steps backwards through the call sequence. At each stage, the    `status`, `header`, and `body`  is returned from each `call` method. From the lambda#call,   the focus returns to the `body#call` method and on that methods completion to `header#call ` and finally the application response is sent back to the web server and closes the circle.

At each of these layers, the middleware can manipulate the  status, header and body fields on the down stream and modify the environment  variables passed up stream to higher middleware layers.

The resulting response from our program is :
status: `200`, header: `{"X-Authorization"=>"secret"}` and body: `" from Body"` 

### Websites
- [https://blog.engineyard.com/2015/understanding-rack-apps-and-middleware](https://blog.engineyard.com/2015/understanding-rack-apps-and-middleware)

- [http://chneukirchen.org/blog/archive/2007/02/introducing-rack.html](http://chneukirchen.org/blog/archive/2007/02/introducing-rack.html)

- [http://faxon.org/2012/04/28/understanding-how-rack-uses-ruby-lambdas](http://faxon.org/2012/04/28/understanding-how-rack-uses-ruby-lambdas)
