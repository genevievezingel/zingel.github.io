---
layout: post
title: "Adding Javascript Libraries to a Rails Project"
description: "Alternative ways of adding JavaScript libraries to a Ruby on Rail's project."
date: 2015-12-12
read: "10 min read"
category: software
permalink: /blog/AWS-lambda/
---

##Alternative ways of adding JavaScript libraries to a Ruby on Rail's project

This blog post will examine three common ways of adding JavaScript libraries to a
Rails project, and why I come out in favour of using `rails-assets` RubyGems.


## Option 1 – Adding the JavaScript file to the Vendor Directory
The JavaScript file is copied into the vendor directory with the file name and path included in the <em>mainifest.js </em>. The files listed, in the `manifest.js ` file, by default, are compressed, uglified and concatenated using Rail's Asset Pipeline.


### Advantages and Disadvantages
This provides a quick and easy way of adding JavaScript libraries to your project. The disadvantage is that you don't have any <em> dependency management </em>, nor is it obvious which version has been
installed.

Upgrading to a later version of a library can be a time consuming process as you have to manually check that the correct dependencies have been included, while simultaneously ensuring that you are not breaking other library's dependencies.



<img src="{{ '/assets/images/blog/rails_javascript_setup/setup_vendor.png' | relative_url }}" alt="VSCode" itemprop="image"  class="u-photo">

##Option 2- Using RubyGems to add JavaScript libraries to Rails

Ruby uses RubyGems for package management. For each RubyGem, the name, version, operating system and any dependencies are specified. To add a RubyGem to your Rail’s project, the RubyGem’s name is added to the <em>Gemfile</em> and the command ` bundle install ` is entered into Terminal. This will install the RubyGem along with any dependencies into your Rail’s project.  

To update a RubyGem, or possibly revert to an older version, the required version is specified in the <em>Gemfile </em> file and the command
`bundle install`is entered into Terminal. This will remove the previous RubyGem and add the newly specified RubyGem along with any dependencies to your Rail’s project

Not just Ruby files but any other type of files can be added to a project using RubyGems. RubyGems is not just for Ruby files!  A number of RubyGems developers have used this to add JavaScript libraries to Rail projects. The most famous example is <em> jquery-rails gem</em> which adds the jQuery JavaScript library to Rails.

The <em>big </em> problem is that you are reliant on the RubyGem's maintainers to keep up-to-date with new releases of the JavaScript library.

##Option-3 Using Bower JavaScript Libraries in Rails (rails-assets.org)

Using rails-assets has all of the advantages of using RubyGems to add JavaScript libraries, but without having to rely on a RubyGem maintainer to keep up to-date with the <em>latest</em> version.  

It works by automatically converting a Bower project to a RubyGem so that it can be included and treated like any other RubyGem.


### What is Bower?

What RubyGems is to Ruby, Bower is to JavaScript. It is an easy and lightweight solution
to specifying dependencies and sharing open source libraries. It has the same benefits as RubyGems with version control and
dependency management.


The magic of <em> Rails Asset</em> is that it automatically converts the Bower package into a RubyGem with the JavaScript and CSS being
made available to Rail’s Asset Pipeline.


To understand what is going on lets go through an example:

###Creating a Bower package 

Once you have installed <em>NPM</em> type the following into Terminal.
`npm install -g bower`.

In order to install <em>NPM</em> - a package manager for Node - requires that you first install <em> Node</em>. There are a number of posts that show you how to install <em>Node</em> - see appendix

Create your Bower package by typing into the Terminal the following:

~~~
bower init
~~~

This takes you through a couple of steps to define your Bower Package. You will need to specify a unique project name and the location of the git repository.


Once the Bower package has been created then it can be registered so that it is listed in the Bower’s directory.


To register, type the following into Terminal. My dummy project is called <em> cezDemoBower</em>
~~~
bower register  cezDemoBower git@github.com:chrisZingel/cezDemoBower.git
~~~

You can use the following command in Terminal to check that the package has been correctly setup

~~~
bower info cezDemoBower
~~~

###Using Asset-Rails
To use this Bower Library in your rails application

Include the following in your Gemfile

~~~ ruby
source 'https://rails-assets.org' do
  gem "rails-assets-cezDemoBower"
end
~~~

This will make a request from rails-asset.org for the RubyGem
<strong>rails-assets-cezDemoBower</strong> which it will then fetch from the <em> Bower  Library</em> <strong>cezDemoBower</strong> Bower
Project and convert it in to a RubyGem.  The JavaScript and CSS files, included in the Bower project, will become available to the
Rail’s Asset Pipeline and available to your project.  Note: if a Bower project has any dependencies then they will be included in as a <em>rails asset gem </em>. Neat hey!


## In Summary
Using <em>rails-assets</em> allows you to include any of Bower’s JavaScript libraries directly into your Rails applications. It is a clever way of leveraging existing packages in Bower with all of the advantages of using RubyGems.


##Inspiration
- <a href ="https://github.com/rails-assets/rails-assets/">Convert Bower package into RubyGem using CLI</a>
- <a href ="https://github.com/chrisZingel/cezDemoBower">Demo Bower package</a>
- <a href ="http://www.gotealeaf.com/blog/rails-asset-pipeline-best-practices"> About asset pipeline </a>
- <a href ="http://bower.io/search/">Search available Bower Libary </a>
- <a href ="https://rails-assets.org">Rails Assets </a>
- <a href="https://www.npmjs.com/">NPM- is a packaged manager for JavaScript</a>
