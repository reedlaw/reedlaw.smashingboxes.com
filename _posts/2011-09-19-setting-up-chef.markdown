---
layout: post
title: Setting Up Chef
categories:
- Rails
- Chef
---

{{ page.title }}
================

![chef logo](/images/chef.png "Chef")

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

<span class="label success">Success</span> That should show you a list of your organization's nodes.


Import Cookbooks
----------------
Cookbooks are collections of recipes. Importing them from [Opscode's
community cookbooks repository](http://community.opscode.com/cookbooks) is as easy as:

    knife cookbook site vendor [cookbook-name]

<span class="label warning">Warning</span> This command will pull down a git repository containing the cookbook
and then merge it with `master`. Be careful if you were working in another
branch!

<span class="label important">Important</span> Make the `cookbooks` directory in the root of your app and commit before pulling
down any cookbooks. Be sure to do `touch cookbooks/.gitkeep` before
committing to make sure git keeps that directory.

Assuming we are going to set up a Rails server with Unicorn and a
HAProxy load balancer, let's start with the following cookbooks:

    knife cookbook site vendor application
    knife cookbook site vendor apt
    knife cookbook site vendor git
    knife cookbook site vendor haproxy
    knife cookbook site vendor mongodb
    knife cookbook site vendor users

Now you need to upload the cookbook to your chef server:

    knife cookbook upload -a

<span class="label">New</span> I had some trouble with the mongodb
cookbook. There was a syntax error so I forked it and submitted a pull
request. The cookbook author accepted the request but the community
cookbooks repo was not updated. So I used the
[knife-github-cookbooks
gem](https://github.com/websterclay/knife-github-cookbooks) to install
the latest cookbook from github:

    knife cookbook github install edelight/cookbooks 

Add Roles
---------
A little terminology:
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
project. Then we can add files such as `roles/base.rb`:

    name "base"
    description "Base role applied to all nodes."
    run_list(
      "recipe[users::sysadmins]",
      "recipe[sudo]",
      "recipe[apt]",
      "recipe[git]"
    )
    override_attributes(
      :authorization => {
        :sudo => {
          :users => ["your_name"],
          :passwordless => true
        }
      }
    )

<span class="label important">Important</span> See the following
section on data bags to set up the users and change the `your_name`
value accordingly.

We need to add this to our chef server, just like we added the 
cookbooks above:

    knife role from file roles/base.rb

<span class="label notice">Notice</span> Roles can also be created
with the `knife` tool, as JSON objects, or through chef server's
management interface.

Data Bags
---------
Before we deploy, we need to have a look at our `users` cookbook. [The
documentation](https://github.com/opscode/rails-quick-start/tree/master/cookbooks/users)
states that you need to create a data bag for users:

    knife data bag create users

We'll need a directory to put the user file in (do this in the root of
your Rails app):

    mkdir -p data_bags/users

Now we can create a user. Open `data_bags/users/your_name.json` and
make it look like this:

    {
      "groups": "sysadmin",
      "ssh_keys": "ssh-rsa AAAAB3Nz...",
      "id": "your_name",
      "shell": "/bin/bash"
    }

<span class="label important">Important</span> Be sure to change
`your_name` to the user name you actually want to use and paste the
user's public SSH key into the ssh_keys value.

We'll need another data bag for our application,
call it `data_bags/rails_demo.json`

    {
      "id": "rails_demo",
      "server_roles": [
        "rails_demo"
      ],
      "type": {
        "rails_demo": [
          "rails",
          "unicorn"
        ]
      },
      "packages": {
        "libxml2-dev": "",
        "libxslt1-dev": "",
        "libsqlite3-dev": ""
      },
      "gems": {
        "bundler": "1.0.18",
      },
      "repository": "git://github.com/reedlaw/chef-tutorial.git",
      "deploy_to": "/srv/rails_demo",
      "deploy_key": "",
      "owner": "nobody",
      "group": "nogroup",
      "force": {
        "_default": false
      },
      "migrate": {
        "_default": true
      }
    }

Upload this data bag with:

    knife data bag from file apps data_bags/rails_demo.json

Setting up EC2
--------------

Sign into your Amazon AWS web console and create a security group, `single_instance_production`, and open the following ports:

* 22 - ssh
* 80 - haproxy load balancer
* 22002 - haproxy administrative interface
* 8080 - unicorn Rails application

First install some dependencies:

    gem install net-ssh net-ssh-multi fog highline --no-rdoc --no-ri

Now you should install the [knife-ec2 gem](https://github.com/opscode/knife-ec2) in order to provision EC2
instances:

    gem install knife-ec2

Bootstrapping Chef with Ruby 1.9.2
----------------------------------
Turns out that chef installs ruby 1.8.7 by default. To fix this we
need to add a 'bootstrap' file that gets run when the server is
provisioned ([thanks to Adam's blog for the
tip](http://www.oneadam.net/?p=246)).

Create a file, `bootstrap/ubuntu10.04-ruby1.9.2.erb` with the
following:

{% highlight ruby %}
bash -c '
if [ ! -f /usr/local/bin/chef-client ]; then
  apt-get update
  sudo apt-get install -y build-essential zlib1g zlib1g-dev openssl libssl-dev
  cd /tmp
  wget ftp://ftp.ruby-lang.org//pub/ruby/1.9/ruby-1.9.2-p290.tar.gz
  tar xvf ruby-1.9.2-p290.tar.gz
  cd ruby-1.9.2-p290
  ./configure
  make
  sudo make install
  sudo gem update --system

  gem update
  gem install ohai --no-rdoc --no-ri --verbose
  gem install chef --no-rdoc --no-ri --verbose <%= bootstrap_version_string %>
fi 

mkdir -p /etc/chef

(
cat <<'EOP'
<%= validation_key %>
EOP
) > /tmp/validation.pem
awk NF /tmp/validation.pem > /etc/chef/validation.pem
rm /tmp/validation.pem

(
cat <<'EOP'
<%= config_content %>
EOP
) > /etc/chef/client.rb

(
cat <<'EOP'
<%= { "run_list" => @run_list }.to_json %>
EOP
) > /etc/chef/first-boot.json

/usr/local/bin/chef-client -j /etc/chef/first-boot.json'
{% endhighlight %}

This script will run as soon as the server is up and compile ruby from source.

Launch Instance
---------------

For this example we are going to have a node named `chef-tutorial`.

This command should launch a server and configure it with Hosted Chef:

    knife ec2 server create --node-name ctr --groups single_instance_production --image ami-7000f019 --flavor m1.small --distro ubuntu10.04-ruby1.9.2 --ssh-key ampms -i ~/.ec2/ampms -x ubuntu --run-list 'role[base],role[rails_demo]'

You should see a bunch of commands executing on the server and finally
something like this:

    Instance ID: i-48b83b28
    Flavor: m1.small
    Image: ami-7000f019
    Region: us-east-1
    Availability Zone: us-east-1a
    Security Groups: single_instance_production
    SSH Key: ampms
    Root Device Type: instance-store
    Public DNS Name: ec2-50-19-193-102.compute-1.amazonaws.com
    Public IP Address: 50.19.193.102
    Private DNS Name: ip-10-80-21-131.ec2.internal
    Private IP Address: 10.80.21.131
    Environment: _default
    Run List: role[base]

You should now be able to ssh in and see that everything is set up
right:

    ssh -i ~/.ec2/private_key ubuntu@ec2-50-19-193-102.compute-1.amazonaws.com

Troubleshooting
---------------

Files for this tutorial are
[here](https://github.com/reedlaw/chef-tutorial).

