---
layout: post
title: Setting Up Chef
---

{{ page.title }}
================

![chef logo](/images/chef.png "Chef")

<span class="label success">New</span> 

<span class="label important">Important</span>
<span class="label notice">Notice</span>

Chef is like Rails for systems integration. It's a framework designed
to create infrastructure as code.

There are three parts to chef: server, client, and the command line
tools. For the server we are going to use Hosted Chef which is free
for up to 5 nodes. 

First let's get the client set up.

Client Setup
------------

Install the chef gem:

    gem install chef

Now create a `.chef/knife.rb` file in your project:

{% highlight ruby %}
current_dir = File.dirname(__FILE__)
user = ENV['OPSCODE_USER']
log_level                :info
log_location             STDOUT
node_name                user
client_key               "#{ENV['HOME']}/.chef/#{user}.pem"
validation_client_name   "#{ENV['ORGNAME']}-validator"
validation_key           "#{ENV['HOME']}/.chef/#{ENV['ORGNAME']}-validator.pem"
chef_server_url          "https://api.opscode.com/organizations/#{ENV['ORGNAME']}"
cache_type               'BasicFile'
cache_options( :path => "#{ENV['HOME']}/.chef/checksums" )
cookbook_path            ["#{current_dir}/../cookbooks"]

knife[:aws_access_key_id]     = ENV['AWS_ACCESS_KEY_ID']
knife[:aws_secret_access_key] = ENV['AWS_SECRET_ACCESS_KEY']
knife[:identity_file]         = "#{ENV['HOME']}/.ec2/ampms"
knife[:aws_ssh_key_id]        = ENV['AWS_SSH_KEY_ID']

knife[:availability_zone] = "us-east-1a"
knife[:region]            = "us-east-1"
knife[:aws_image_id]      = "ami-81b275e8"
{% endhighlight %}

Notice the environment variables being called there? Yes, we need to
set them up. 

Setting Up Our Environment
--------------------------

[This is a pretty good
guide](http://help.opscode.com/kb/start/2-setting-up-your-user-environment)
on signing up with Opscode for Hosted Chef and downloading the required `.pem`
files.

So now that you've set up your user and organization on Hosted Chef,
you need to add them to your environment. `.bashrc` should be
a good place to put this:

    export ORGNAME=your_org
    export OPSCODE_USER=your_name
    export AWS_ACCESS_KEY_ID=your_key
    export AWS_SECRET_ACCESS_KEY=your_secret_key
    export AWS_SSH_KEY_ID=name_of_your_amazon_pub_key_file

Let's check that everything's good:

    knife node list

That should show you a list of your organization's nodes.

Import Cookbooks
----------------
Cookbooks are collections of recipes. Importing them from [Opscode's
community cookbooks repository](http://community.opscode.com/cookbooks) is as easy as:

    knife cookbook site vendor [cookbook-name]

<span class="label warning">Warning</span> This command will pull down a git repository containing the cookbook
and then merge it with `master`. Be careful if you were working in another
branch!

Now you need to upload the cookbook to your chef server:

    knife cookbook upload [cookbook-name]

Add Roles
---------
Now for a little terminology:
<dl>
  <dt>Node</dt>
  <dd>One server instance</dd>
  <dt>Environment</dt>
  <dd>Chef determines which recipes to run based on which role and
  environment you've chosen. So a role 'webserver' can have different
  recipes for a 'production' environment and a 'staging'
  environment</dd>
  <dt>Role</dt>
  <dd>Tags representing related functionality</dd>
</dl>
So to create a role we create a `roles` directory at the root of our
project. Then we can add files such as `roles/web.rb`:

    name "web"
    description "web server"
    run_list(
      "recipe[nginx]",
      "recipe[haproxy]"
    )
We need to add this to our chef server, just like we added the 
cookbooks above:

    knife role from file web.rb

