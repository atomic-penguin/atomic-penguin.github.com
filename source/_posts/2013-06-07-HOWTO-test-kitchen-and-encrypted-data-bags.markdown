---
layout: post
title: "HOWTO test-kitchen and encrypted data bags"
date: 2013-06-07 15:58
comments: true
categories: test-kitchen opschef "integration testing" "data_bags" "encrypted data bags" HOWTO
---

I have been playing around with [test-kitchen](https://github.com/opscode/test-kitchen)
more recently.  One of the community members suggested I embed an encrypted data_bag
into the certificates cookbook for integration testing with test-kitchen.

Following, is the process used to set up test-kitchen for encrypted data_bag
use.  This documentation might be useful to others trying to integrate encrypted
data_bag testing into their cookbooks.

### Warning

Encrypted data_bag files placed in `test/integration` should not be used in production.
Files include a self-signed "snake oil" certificate/key and an encrypted\_data\_bag\_secret file
which are not secure to use beyond testing purposes.

### Initialize test-kitchen

Run the following to create a `.kitchen.yml` file in your cookbook directory.

    $ cd <cookbook dir>
    $ kitchen init

This `kitchen init` command will yield a file which looks like this:

    ---
    driver_plugin: vagrant
    driver_config:
      require_chef_omnibus: true

    platforms:
    - name: ubuntu-12.04
      driver_config:
        box: opscode-ubuntu-12.04
        box_url: https://opscode-vm.s3.amazonaws.com/vagrant/opscode_ubuntu-12.04_provisionerless.box
    - name: ubuntu-10.04
      driver_config:
         box: opscode-ubuntu-10.04
         box_url: https://opscode-vm.s3.amazonaws.com/vagrant/opscode_ubuntu-10.04_provisionerless.box
    - name: centos-6.4
      driver_config:
        box: opscode-centos-6.4
        box_url: https://opscode-vm.s3.amazonaws.com/vagrant/opscode_centos-6.4_provisionerless.box
    - name: centos-5.9
      driver_config:
        box: opscode-centos-5.9
        box_url: https://opscode-vm.s3.amazonaws.com/vagrant/opscode_centos-5.9_provisionerless.box

    suites:
    - name: default
      run_list: ["recipe[certificate::default]"]                                                                                                                                    
      attributes: {}

### Data bag and secret file setup

The test-kitchen initialization should create a `test/integration/default` directory.
As of test-kitchen pull request #124, cookbook files are aggressively filtered from being
copied to the chef-solo working directory.  Adding these files to the `suite` section of the
`.kitchen.yml` will ensure those files get copied over for your integration testing.

First, create a throwaway secret to encrypt testing data.

    openssl rand -base64 512 > test/integration/default/encrypted_data_bag_secret

Second, create an encrypted data bag with test data.  Apparently there is also
a [knife-solo_data_bag](https://github.com/thbishop/knife-solo_data_bag) knife
plugin which can be used alternatively.

    knife data bag create certificates test --secret-file test/integration/default/encrypted_data_bag_secret

Copy and paste the encrypted data bag object into a file.

    # Make a data_bag directory
    mkdir -p test/integration/default/data_bags/certificates

    # Copy the encrypted json output
    knife data bag show certificates test -Fj

    # Paste the encrypted json output into a json file
    cat > test/integration/default/data_bags/certificates/test.json

### Configure test-kitchen

Edit the suites section in the `.kitchen.yml` and add `encrypted_data_bag_secret_key_path`
and `data_bag_path`.

Default generated configuration.

    suites:
    - name: default
     run_list: ["recipe[certificate::default]"]
      attributes: {}

Configuration for test-kitchen with encrypted data bags.

    suites:
    - name: default
      data_bag_path: "test/integration/default/data_bags"
      encrypted_data_bag_secret_key_path: "test/integration/default/encrypted_data_bag_secret"
      run_list: ["recipe[certificate::default]"]
      attributes: {}

That should be all you need to run test-kitchen integration tests
with encrypted data bags.


