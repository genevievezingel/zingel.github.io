---
layout: post
title: "Accidentally applying the Strategy Pattern"
description: "I accidentially adopting the <strong>stratgey pattern</strong> for a recent project had me working on task extracting lists from a number of  websites. My initial solution was to have a single class. It didn’t take long before it become verbose, with a number of branching <code>if/else</code> statements depending on the site being extracted. While my solution worked OK for one or two sites. I wasn’t going to be maintainable for  100’s of sites."
date: 2014-03-11
read: "6 min read"
category: software
permalink: /blog/accidential-strategy-pattern/
---

### Accidental strategy pattern
A recent project had me working on task extracting lists from a number of  websites. My initial solution was to have a single class. It didn’t take long before it become verbose, with a number of branching `if/else` statements depending on the site being extracted. While my solution worked OK for one or two sites. I wasn’t going to be maintainable for  100’s of sites.

#### There two types of code:
In examining the code, the code could be separated into:
- Re-useable code
- Detailed/ Unique  code; specific to the website being scrapped.

The `re-useable code` captures the patterns that needs to be followed in order to extract the listings from the site. These steps or pattern include: interacting with the HTML response/ request cycle, iterating through the listings and saving the results into the database.

The second type of code is `particular to the website` being scrapped and would be different for each website. For example, the code used to identify each record in the HTML and to parse the resulting text.

The solution that I had stumbled upon was to separate out the re-usable code from the website specific code. This is done using  `procs` to store the website specific code (or rules). I had accidentally adopted the strategy pattern.

#### An example of the Strategy Pattern

~~~ ruby
class Person
  attr_accessor :name

  def initialize(&rules)
	instance_exec(&rules)
  end
end

Person.new do
   # the rules go here
end
~~~

The `strategy pattern` separates the code that changes "rules" from the part that is re-used. In the above case, the re-used portion of code is contained in the `Person` class. The strength of this pattern is that you can change the rules at run-time and it enhances clarity as the intent of the code is not distorted by the particular implementation. The implementation of this pattern uses `instance_exec` to execute the rules - but more on that later.


### Using lambdas and instance_exec

I didn’t have a good handle on lambdas. It seemed strange to me that the context was assigned when it was created rather than on execution. Thankfully, time spend with JavaScript, has cured me of this, as its very similar to JavaScript's concept of closures. The big difference to lambdas is how `this` and `self` is assigned. In JavaScript, `this` is determined by the calling object. However, in Ruby's lambdas, ` self` is assigned when the lambda was created.

 Take a look at the following example:

~~~ ruby
@some_variable ='some variable'
HTML_FORMATER = lambda do
  puts @some_variable
end

class Report
  def initialize(&format)
	@some_variable ='this is variable in Report'
	format.call()
  end
end

Report.new(&HTML_FORMATER)
=> some variable
~~~

The `@some_variable` references the value  "some variable" as this was the context when the block was created. However, if we want to change the content to refer to the object calling the method, this can achieved using `instance_exec`.

 To see this in action

~~~ ruby
class Report
  def initialize(&format)
	@some_variable ='this is some variable in Report'
	instance_exec(&format)
  end
end


Report.new(&HTML_FORMATER)
=>this is some variable in Report
~~~

 The `@some_variable` references the value *”this is some variable in Report”* . The context used is the Report instance, which is what I am after.

### My code as a final example:
 I have implemented this as a `rack task` so that I can run it daily using CRON. I have simplified it to show the overall strategy.

~~~ ruby
# encoding=UTF-8
require 'rubygems'
require 'faraday'
require 'nokogiri'
require 'open-uri'
require 'yaml'
require 'ostruct'

# rake task to

namespace :api do
  desc 'Search Engine Ranking'
  task :search_engine_ranking => :environment do
	def env_die
	  abort 'Fatal error: You need to specify a valid ENV.'
	end
	# Ensure that ENV is set to something
	if ENV['RAILS_ENV'].nil?
	  env_die
	end

	class SearchEngine
	  attr_accessor :params, :conn, :response, :results,
		  :search_engine_request,:count

	  def initialize(keywords, base_url)
		@params = OpenStruct.new
		@count =0
		@params.base_url = base_url
		@params.path ='/search?q=' + keywords.join('+')
		@conn = Faraday.new(:url => @params.base_url) do |faraday|
		  faraday.response :logger                  # log requests to STDOUT
		  faraday.request  :url_encoded             # form-encode POST params
		  faraday.adapter  Faraday.default_adapter  #  Net::HTTP request
		end
		yield(@params)
		self.instance_exec(params, &params.rule_for_setup)
				unless params.rule_for_setup.nil?
		@search_engine_request =
			  SearchEngineRequest.create(keywords: keywords.join('+'),
										 site: base_url)
	  end

	  def get_action(path=params.path)
		puts 'request on #{path}'
		@response = conn.get path do |response|
		  response.headers['Cookie'] = params.cookie
				unless params.cookie.nil?
		end
	  end

	  def extract_records
		doc = Nokogiri::HTML response.body
		doc.css(self.instance_exec(&params.rule_to_indentify_records))
	  end

	  def process_results
		@results =[]
		extract_records.each do |record|
		  @count +=1
		  match, url, path =
			 self.instance_exec(record, &params.rule_to_extract_record)
		  @results << {order_id: @count, url: url,
			  path: path, match?: match } unless match.nil?
		end
		save_results
	  end

	  def save_results
		results.each do | record|
		  puts record
		  search_engine_request.search_engine_results.create(
							{ rank_order: record[:order_id],
							 match: record[:match?],
							 path: record[:path][0..100],
							 url: record[:url][0..100] })
		end
	  end

	  def loop_through_the_pages
		0.step(100,10).each do |start|
		  get_action(params.url_first_part + start.to_s + params.url_last_part)
		end
		process_results
	  end
	end

	url = 'https://www.xxxx.com'
	keywords =['ecommerce', 'spree']

	search_engine = SearchEngine.new(keywords,url) do |params|

	  params.rule_to_indentify_records = lambda{ '#results li'}

	  params.rule_to_extract_record = lambda{ |record|
		search_url = begin
		   record.css('h3 > a')[0].attribute_nodes.
			   find{|i| i.name='href'}.value
		 rescue
		   ''
		 end
		match = begin
		   record.text.match(%r{xx -your expr xxxxxx}).nil? ? false : true
		 rescue
		   false
		 end
		url_path_match = search_url.
			   match(/.*http:\/\/(?<url>.*?)\/(?<path>.*)/)
		url_path_match.nil? ? [] :
			  [match,url_path_match['url'], url_path_match['path']]
	  }

	  params.rule_for_setup = lambda{ |p|
		response = get_action(p.path)
		p.cookie = response.headers['set-cookie']
		doc = Nokogiri::HTML response.body
		raw_url =doc.css('.sb_pag li a')[2].attribute_nodes.
		   find{|i| i.name =='href'}.value
		match_url=raw_url.match(/(?<first_part>.+?first=)\d+(?<last_part>.*)/)
		p.url_first_part, p.url_last_part =
			[match_url['first_part'], match_url['last_part']]
	  }
	end

	search_engine.loop_through_the_pages
  end
end
~~~

<a href="https://gist.github.com/chrisZingel/9593454">view the gist</a>

### Links
  - <a href = "http://designpatternsinruby.com/"> russ olsen's design pattern book</a>
  - <a href="https://gist.github.com/chrisZingel/9593454">view the gist </a>
