---
layout: post
title: "Multi-knife HOWTO"
date: 2013-04-25 20:26
comments: true
categories: HOWTO, knife, opschef, chef, devops
---

Overview
========

  Here is an example shared configuration for knife, the command and control utility which ships with Opscode Chef.  You can drop this off in your `chef-repo/.chef/` directory, and multiple developers can use the same knife configuration to interact with more than one Chef server, or the Opscode platform.

  By using Bash functions and environment variables we can change the chef server, which knife is configured to use, on the fly.

<!-- more -->

**NOTE**: knife will probably ignore your `~/.chef/knife.rb` once you begin using a shared knife configuration.

Preparation
===========

  So that we can interact with `api.opscode.com`, and an internal Open Source Chef server, you will need to name your keys according to your Chef server username, Opscode username, and Opscode platform organization name.  Assuming you already have a `~/.chef` directory, we will organize it like so.

  1. Re-name your Open Source chef-server private key according to your username on the chef-server system.  For example, move your chef-server private key to `~/.chef/corporate-username.pem`.
  2. Re-name your Opscode API key according to your Opscode platform username, for example I would move my Opscode key to `~/.chef/atomic-penguin.pem` to correspond to my own username.
  3. Re-name your Organization validation key so that it corresponds to your organization's name on the Opscode platform.  For example, move your Opscode organization validation key to ~/.chef/companyname-validator.pem.

Environment in ~/.bashrc
======================

  Somewhere near the end of my `~/.bashrc`, I am going to set 4 environment variables which will later be referenced in my shared knife configuration.  I am also going to use a couple of Bash functions to switch context between multiple servers.

Environment variables
---------------------

The variables are as follows.

  1. `OPSCODE_USER` - Set to your primary username.  This should be either your internal chef-server username or Opscode platform username.
  2. `ORGNAME` - Should be set to "chef" for an internal chef-server.  Set this as your Opscode platform orgname, if using that as your primary Chef server.
  3. `COOKBOOK_COPYRIGHT` - Set to your full name.
  4. `COOKBOOK_EMAIL` - Set to your e-mail address.

Bash functions
--------------

  I have two knife related Bash functions in my `~/.bashrc` file.  I use a function called "ginsu" to interact with the Opscode API.  Then I use the "knife-reset" function to restore default behavior to interact with our internal Chef server.  This is just an example, you could use the ginsu function as a template.  This way you could have an "opscode-knife" function, or maybe "prod-knife", "dev-knife", or "preprod-knife" to correspond to different servers or environments.

  The `knife-reset` function simply sets my `OPSCODE_USER` variable equal to my internal Open Source chef-server username, and `ORGNAME` equal to "chef".  The `knife-reset` function then exports the variables so my knife configuration gets reset back to my primary Chef server.

  The `ginsu` function sets the `OPSCODE_USER` variable equal to my Opscode platform username, and `ORGNAME` equal to my Opscode organization name.  The `ginsu` function then exports the variables and passes any parameters to the knife command.  This way I can do a `ginsu cookbook site share yumrepo "Package Management"` to upload my yumrepo cookbook to the Opscode Community Cookbook site.

Knife configuration
===================

  This is the actual knife configuration file, which we can drop off in the `chef-repo/.chef` directory.  I will explain the important parts, so you can modify this to fit your own needs easily.  Note, once you drop this off in your `chef-repo`, your `~/.chef/knife.rb` will no longer work.

Set a local Ruby variable `user` equal to the `OPSCODE_USER` or the environment variable `USER`.

```bash
user = ENV['OPSCODE_USER'] || ENV['USER']
```

Switching back and forth between different keys and user names occurs by changing, and exporting, the `OPSCODE_USER` and `ORGNAME` Bash environment variables.

```bash
client_key               "#{ENV['HOME']}/.chef/#{user}.pem"
validation_client_name   "#{ENV['ORGNAME']}-validator"
validation_key           "#{ENV['HOME']}/.chef/#{ENV['ORGNAME']}-validator.pem"
```

I used an if/then/else block in my shared knife.rb to switch the URL of my Chef server based on the `ORGNAME`.  You could use a `CHEF_SERVER` environment variable in your `~/.bashrc` if that would work better for you.  Obviously you'll want to change the hardcoded `corporate` string to match your Opscode platform orgname, and also edit the `chef_server_url` to point another Chef server.

```ruby
if ENV['ORGNAME'] == 'corporate'
  chef_server_url "https://api.opscode.com/organizations/#{ENV['ORGNAME']}"
elsif ENV['ORGNAME'] == 'chef'
  chef_server_url 'http://chef.example.com:4000'
end
```

If you commit all your Community Cookbooks under corporate copyright rather than individual developer names, then you might want to hardcode the company name and a generic development team e-mail address.  This way, your developers don't have to set `COOKBOOK_COPYRIGHT` or `COOKBOOK_EMAIL`.  The command `knife cookbook create <cookbook name>` will fill in the company name as the cookbook maintainer, and copyright owner in that case.

```ruby
cookbook_license         'apachev2'
cookbook_copyright ENV['COOKBOOK_COPYRIGHT'] || 'Corporate, Inc.'
cookbook_email ENV['COOKBOOK_EMAIL'] || 'cookbooks@example.com'
```

Reference ~/.bashrc
===================

```bash
OPSCODE_USER=chef-server-username
ORGNAME=chef
COOKBOOK_COPYRIGHT='My Full Name'
COOKBOOK_EMAIL='dev@example.org'
AWS_ACCESS_KEY_ID="BlahBlahBlah"
AWS_SECRET_ACCESS_KEY="BlahBlahBlahBlah"
export OPSCODE_USER ORGNAME COOKBOOK_COPYRIGHT COOKBOOK_EMAIL AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY

function knife-reset {
  OPSCODE_USER=chef-server-username
  ORGNAME=chef
  export OPSCODE_USER ORGNAME
}

function ginsu {
  OPSCODE_USER=community-username
  ORGNAME=community-organization
  export OPSCODE_USER ORGNAME
  knife "$@"
}
```

Reference chef-repo/.chef/knife.rb
==================================

```ruby
current_dir = File.dirname(__FILE__)
user = ENV['OPSCODE_USER'] || ENV['USER']
log_level                :info
log_location             STDOUT
node_name                user
client_key               "#{ENV['HOME']}/.chef/#{user}.pem"
validation_client_name   "#{ENV['ORGNAME']}-validator"
validation_key           "#{ENV['HOME']}/.chef/#{ENV['ORGNAME']}-validator.pem"
if ENV['ORGNAME'] == 'marshall'
chef_server_url "https://api.opscode.com/organizations/#{ENV['ORGNAME']}"
elsif ENV['ORGNAME'] == 'chef'
chef_server_url 'http://chef.example.org:4000'
end
cache_type               'BasicFile'
cache_options( :path => "#{ENV['HOME']}/.chef/checksums" )
cookbook_path            ["#{current_dir}/../cookbooks", "#{current_dir}/../site-cookbooks"]
role_path ["#{current_dir}/../roles"]
cookbook_license         'apachev2'
cookbook_copyright ENV['COOKBOOK_COPYRIGHT'] || 'Corporate, Inc.'
cookbook_email ENV['COOKBOOK_EMAIL'] || 'corporate@example.com'
knife[:aws_access_key_id] = ENV['AWS_ACCESS_KEY_ID']
knife[:aws_secret_access_key] =  ENV['AWS_SECRET_ACCESS_KEY']
knife[:identity_file] = "#{ENV['HOME']}/.ssh/id_rsa"
knife[:aws_ssh_key_id] = ENV['OPSCODE_USER'] || ENV['USER']
knife[:availability_zone] = "us-east-1a"
knife[:region] = "us-east-1"
```

Credits
=======

This was partly adapted from Stephen Nelson-Smith's (@LordCope) book Test-Driven Infrastructure with Chef (http://shop.oreilly.com/product/0636920020042.do).
