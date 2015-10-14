---
layout: post
title: "Using Chef-Provisioning on the Rackspace Cloud"
date: 2015-10-14 00:01
comments: true
author: Martin Smith
bio: >
  Damien is a software developer at Rackspace that is currently working on
  the open source Repose project. He graduated from the University of Texas at
  Austin in December of 2013, and has been delivering fanatical results at
  Rackspace ever since!
published: true
categories:
- Cloud Servers
- Public Cloud
- Chef
- Configuration Management
authorIsRacker: true
authorAvatar: http://gravatar.com/avatar/9ad2a5355d8cfa842e24b7a4322b2535/?s=64&d=mm
---

Many Rackspace customers use configuration management tools to manage their
cloud servers and other cloud infrastructure. But some configuration management
tools are now also enabling users to not just manage cloud servers, but actually
create the servers as well. As part of the
[DevOps Automation Service](http://rackspace.com/devops), I most often work with
a popular configuration management tool called [Chef](http://www.chef.io).
Working in conjunction with Chef, there is also a project called [Chef Provisioning](https://github.com/chef/chef-provisioning) that can use APIs
to build cloud servers and Docker containers on many different providers (AWS,
Azure, OpenStack, etc). Chef Provisioning can then bootstrap the
new instance or container and begin configuration management tasks.

I'd like to introduce one specific driver for Chef Provisioning, the
[chef-provisioning-fog](https://github.com/chef/chef-provisioning-fog) driver.
This driver can be used with Chef Provisioning to build Rackspace cloud servers
with simple Chef recipes. I will show an example of how to use these tools to
automate building cloud servers, and provide you with [an example you can try
locally](https://github.com/martinb3/chef-provisioning-rackspace-example).

<!-- more -->

# Pre-requisites

Before anything else, we need to be sure you have a few key pieces of
information. Specifically, we will need:

- **Rackspace username & API key**: You will need a cloud account at Rackspace to run this example locally. For our examples, the username will be `my_rackspace_user` and the API key will be `api_key_for_user`. You can read about [how the fog library uses these settings](http://fog.io/about/getting_started.html) or [sign up for a new Rackspace Cloud account](https://cart.rackspace.com/cloud/) today.

- **Region**: We will build our examples in `DFW`, but you may choose [any valid Rackspace Public Cloud region](http://www.rackspace.com/knowledge_center/article/about-regions).

- **Server Image ID**: This will determine what image your new server is booted from, is how we must choose between Linux, Windows, and the variants of each. For this example, we have selected `fabe045f-43f8-4991-9e6c-5cabd617538c`, which is a CentOS 6 (PVHVM) image. You may read about how to find the full list of valid image IDs in our [documentation on listing cloud server flavors](http://docs.rackspace.com/servers/api/v2/cs-devguide/content/List_Flavors-d1e4188.html).

- **Server Flavor**: This will determine the size and performance characteristics of the newly created server. We are going to use `general1-2` in this example, which represents a General Purpose server with 2GB of RAM and 40GB of disk.  You may find the full list of valid values in our [documentation on cloud server flavors](http://docs.rackspace.com/servers/api/v2/cs-devguide/content/server_flavors.html#supported_flavors_table).

- **SSH Keypair**: This keypair will allow Chef Provisioning to authenticate to newly created servers and begin the process of bootstrapping them with Chef. We will refer to this as `id_rsa_example`, but you will want go generate your own keypair before proceeding.

You'll want to make note of all of these items so that you can configure your `knife.rb` file as described in the next section.

# Configuration

You should create or update your `knife.rb` file with the following settings. More information on where this file is located [can be found in the documentation](https://docs.chef.io/config_rb_knife.html), but if this is a new workstation and you've never used chef or knife before, you can place this file at `~/.chef/knife.rb`.

```
driver 'fog:Rackspace'
driver_options :compute_options => {
  :rackspace_username => 'my_rackspace_user',
  :rackspace_api_key  => 'api_key_for_user',
  :rackspace_region => 'dfw' # could be 'org', 'iad', 'hkg', etc
}
```

As you can see, this file just contains some of the configuration options we wrote about in the previous section.

# Creating a server

I've created [a repository with example code](https://github.com/martinb3/chef-provisioning-rackspace-example), but first, I'd like to step through each part of the Chef recipe that I'm going to show today. It will first require the necessary dependencies, create a keypair, set options, and finally create the new cloud server and bootstrap Chef on it.

These lines ensure that Chef Provisioning and the fog driver are both loaded for the recipe to use.
```
require 'chef/provisioning'
require 'chef/provisioning/fog_driver/recipe_dsl'
```

This [creates or updates a keypair](http://docs.rackspace.com/servers/api/v2/cs-devguide/content/CreateKeyPair.html) within Rackspace's Compute API, so we can reference the keypair later when building servers.
```
fog_key_pair 'example_id_rsa'
```

This is part of the DSL provided by Chef Provisioning. We use `with_machine_options` to set various configuration elements; here, we are ensuring we get a 2gb General instance flavor with a CentOS 6 (PVHVM) image, and that the keypair we created earlier is injected upon creation.

```
with_machine_options({
  :bootstrap_options => {
    :flavor_id => 'general1-2', # required size of server
    :image_id  => 'fabe045f-43f8-4991-9e6c-5cabd617538c', # required, CentOS 6
    :key_name  => 'example_id_rsa', # keypair
  }
})
```

Finally, we create a server named 'mario', tag it with 'itsa_me' (this is a chef tag), and supply `converge true` to bootstrap the new server with chef.

```
machine 'mario' do
  tag 'itsa_me'
  converge true
end
```

In order to actually run this example, simply run `bundle exec chef-client -z recipes/create.rb` from within the demo repository.

# Destroying a server

As you might expect, destroying a server is almost as simple as creating one was. You simply need to use the same machine resource `mario` and change the action to `:destroy`.

```
machine 'mario' do
  action :destroy
end
```

In order to actually run this example, simply run `bundle exec chef-client -z recipes/destroy.rb` from within the demo repository.

# Wrap-Up (and how to contribute)

While this is just [a simple example](https://github.com/martinb3/chef-provisioning-rackspace-example) of using chef-provisioning-fog to create servers on our Rackspace public cloud, it's by no means the limit of current functionality. Chef Provisioning has features like [creating servers in parallel](https://docs.chef.io/resource_machine_batch.html), [taking an image of an existing server](https://docs.chef.io/resource_machine_image.html), [running commands on the remote server](https://docs.chef.io/resource_machine_execute.html), [creating cloud-agnostic load balancers](https://docs.chef.io/resource_load_balancer.html), and even [provisioning docker containers](https://github.com/chef/chef-provisioning-docker#chef-provisioning-docker) instead of servers. If you're interested in more general examples for other cloud providers, [JJ Asghar has blogged other examples](http://jjasghar.github.io/blog/2015/09/25/chef-provisioning-fog-step-by-step-day-by-day/), using my example repository for some inspiration. I'd highly recommend his blog, as he also has [much more elaborate examples](http://jjasghar.github.io/blog/2015/09/25/chef-provisioning-fog-step-by-step-day-by-day/) of what's possible with Chef Provisioning today.

I highly recommend checking out Chef Provisioning, as well as the drivers for AWS, Azure, fog, docker, and SSH.

If you have any questions, would like to know more, or would like to get
involved in the this effort, please feel free to
[send me an email](mailto:damien.johnson@rackspace.com?subject=WSGI%20Middleware%20On%20The%20JVM)
and/or [check out the Repose project!](https://github.com/rackerlabs/repose)
