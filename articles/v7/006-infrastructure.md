> This issue of Practicing Ruby was a collaboration with Mathias Lafeldt
([@mlafeldt](https://twitter.com/mlafeldt)), an Infrastructure
Developer living in Hamburg, Germany. If Mathias had to choose the one
Internet meme that best describes his work, it would certainly be
_Automate all the things!_ 

For at least as long as Ruby has been popular among web developers, it has also
been recognized as a useful tool for system administration work. Although it was
first used as a clean alternative to Perl for adhoc scripting, Ruby quickly
evolved to the point where it became an excellent platform for large scale 
infrastructure automation projects. 

In this article, we'll explore realistic code that handles various system
automation tasks, and discuss what benefits the automated approach has over 
doing things the old-fashioned way. We'll also see first-hand what it means to treat
"infrastructure as code", and the impact it has on building maintainable systems. 

## Prologue: Why does infrastructure automation matter?

Two massive infrastructure automation systems have been built in 
Ruby ([Puppet][puppet] and [Chef][chef]), both of which have entire open-source
ecosystems supporting them. But because these frameworks were built by and for
system administrators, infrastructure automation is often viewed as a
specialized skillset by Ruby programmers, rather than something that everyone
should learn. This is probably an incorrect viewpoint, but it is one that is
easy to hold without realizing the consequences.

Speaking from my own experiences, I had always assumed that infrastructure
automation was a problem that mattered mostly for large-scale public web
applications, internet service providers, and very complicated enterprise
projects. In those kinds of environments, the cost of manually setting up
servers would obviously be high enough to justify using a 
sophisticated automation framework. But because I never encountered those
scenarios in my own work, I was content to do things the old-fashioned way:
reading lots of "works for me" instructions from blog posts, manually typing
commands on the console, and swearing loudly whenever I broke something. For
things that really matter or tasks that seemed too tough for me to do on my own,
I'd find someone else to take care of it for me.

The fundamental problem was that my system-administration related pain wasn't 
severe enough to motivate me to learn a whole new way of doing things. Because
I never got curious enough about the topic, I didn't realize that infrastructure 
automation has other benefits beyond eliminating the costs
of doing repetitive and error-prone manual configuration work. In particular,
I vastly underestimated the value of treating "infrastructure as code",
especially as it relates to creating systems that are abstract, modular,
testable, understandable, and utterly hackable. Narrowing the problem down to
the single issue of reducing repetitive labor, I had failed to see that
infrastructure automation has the potential to eliminate an entire class of
problems associated with manual system configuration.

To help me get unstuck from this particular viewpoint, Mathias Lafeldt offered
to demonstrate to me why infrastructure automation matters, even if you aren't
maintaining hundreds of servers or spending dozens of hours a week babysitting
production systems. To teach me this lesson, Mathias built a [Chef cookbook][pr-cookbook] to completely automate the process of building an environment suitable for running [Practicing Ruby's web application][pr-web], starting with nothing but a bare Ubuntu
Linux installation. The early stages of this process weren't easy: Jordan and I
had to answer more questions about our system setup than I
ever thought would be necessary. But as things fell into place and
recipes started getting written, the benefits of being able to conceptualize a
system as code rather than as an amorphous blob of configuration files and
interconnected processes began to reveal themselves.

The purpose of this article is not to teach you how to get up and running with
Chef, nor is it meant to explain every last detail of the cookbook that
Mathias built for us. Instead, it will help you learn about the core concepts of
infrastructure automation the same way I did: by tearing apart a handful of real
use cases and seeing what you can understand about them. If you've never used
an automated system administration workflow before, or if you've only ever run
cookbooks that other people have provided for you, this article will give you a
much better sense of why the idea of treating "infrastructure as code" matters.
If you already know the answer to that question, you may still benefit from
looking at the problem from a beginner's mindset. In either case, we have
a ton of code to work our way through, so let's get started!

## A recipe for setting up Ruby 

Let's take a look at how Chef can be used 
to manage a basic Ruby installation. As you can see below, Chef
uses a pure Ruby domain-specific language for defining its recipes,
so it should be easy to read even if you've never worked with
the framework before:

```ruby
include_recipe "ruby_build"

ruby_version = node["practicingruby"]["ruby"]["version"]

ruby_build_ruby(ruby_version) { prefix_path "/usr/local" }

bash "update-rubygems" do
  code   "gem update --system"
  not_if "gem list | grep -q rubygems-update"
end

gem_package "bundler"
```

At the high level, this recipe is responsible for handling the following tasks: 

1. Installing the `ruby-build` command line tool.
2. Using `ruby-build` to compile and install Ruby to `/usr/local`.
3. Updating RubyGems to the latest version.
4. Installing the bundler gem.

Under the hood, a lot more is happening. Let's take a closer look at each
step to understand a bit more about how Chef recipes work.

**Installing ruby-build**

```ruby
include_recipe "ruby_build"
```

Including the default recipe from the [ruby_build cookbook](https://github.com/fnichol/chef-ruby_build) 
in our own code takes care of installing the `ruby-build` command line utility, 
and also handles installing a bunch of low-level packages that are required to compile Ruby 
on an Ubuntu system. But all of this work happens behind the scenes -- we just need 
to make use of the `ruby_build_ruby` command this cookbook provides and the rest will be 
taken care of for us.

**Compiling and installing Ruby**

```ruby
ruby_version = node["practicingruby"]["ruby"]["version"]

ruby_build_ruby(ruby_version) { prefix_path "/usr/local" }
```

In our recipe, the version of Ruby we want to install is not specified
explicitly, but instead set elsewhere using Chef's attribute system.
In the cookbook's [default attributes file][pr-cookbook-attributes], you'll find an entry that
looks like this:

```ruby
default["practicingruby"]["ruby"]["version"] = "2.0.0-p247"
```

Chef has a very flexible and very complicated [attribute management system][chef-attributes], but its main purpose is the same as any configuration 
system: to keep source code as generic as possible by not hard-coding
application-specific values. By getting these values out of the
source file and into well-defined locations, it also makes it
easy to see all of our application-specific configuration 
data at once.

**Updating RubyGems**

```ruby
bash "update-rubygems" do
  code   "gem update --system"
  not_if "gem list | grep -q rubygems-update"
end
```

In this code we make use of a couple shell commands, the 
first of which is obviously responsible for updating RubyGems.
The second command is a guard that prevents the gem update
command from running more than once.

Most actions in Chef have similar logic baked into them to
make sure operations are only carried out when necessary. These 
guard clauses are handled internally whenever there is a well defined 
condition to check for, so you don't need to think about them often.
In the case of shell commands the operation is potentially arbitrary,
so a custom guard clause is necessary.

**Installing bundler**

```ruby
gem_package "bundler"
```

This command is roughly equivalent to typing `gem install bundler` on the
command line. Because we installed Ruby into `/usr/local`, it will be used as
our system Ruby, and so we can use `gem_package` without any additional
settings. More complicated system setups would involve a bit more
code than what you see above, but for our purposes we're able to keep 
things simple.

Putting all of these ideas together, we end up not just with an understanding of
how to go about installing Ruby using a Chef recipe, but also a glimpse
of a few of the benefits of treating "infrastructure as code". As we
continue to work through more complicated examples, those benefits
will become even more obvious.

## A recipe for setting up process monitoring 

Now that we've tackled a simple example of a Chef recipe, let's work through 
a more interesting one. The following code is what we use for installing
and configuring the [God][god] process monitoring framework:

```ruby
include_recipe "practicingruby::_ruby"

gem_package "god"

directory "/etc/god" do
  owner "root"
  group "root"
  mode  "0755"
end

file "/etc/god/master.conf" do
  owner    "root"
  group    "root"
  mode     "0644"
  notifies :restart, "service[god]"

  home     = node["practicingruby"]["deploy"]["home_dir"] 
  god_file = "#{home}/current/config/delayed_job.god"

  content "God.load('#{god_file}') if File.file?('#{god_file}')"
end

cookbook_file "/etc/init/god.conf" do
  source "god.upstart"
  owner  "root"
  group  "root"
  mode   "0644"
end

service "god" do
  provider Chef::Provider::Service::Upstart
  action   [:enable, :start]
end
```

The short story about this recipe is that it handles the following tasks:

1. Installing the `god` gem.
2. Setting up some configuration files for `god`.
3. Registering `god` as a service to run at system boot.
4. Starting the `god` service as soon as the recipe is run.

But that's just the 10,000 foot view -- let's get down in the weeds a bit.

**Installing god via RubyGems**

```ruby
include_recipe "practicingruby::_ruby"

gem_package "god"
```

God is distributed as a gem, so we need to make sure Ruby is installed
before we can make use of it. To do this, we include the Ruby installation
recipe that was shown earlier. If the Ruby recipe hasn't run yet, it will
be executed now, but if it has already run then `include_recipe` will
do nothing at all. In either case, we can be sure that we have a
working Ruby configuration by the time the `gem_package` command is called.

The `gem_package` command itself works exactly the same way as it did when we
used it to install Bundler in the Ruby recipe, so there's nothing new to say
about it.

**Setting up a master configuration file**

```ruby
directory "/etc/god" do
  owner "root"
  group "root"
  mode  "0755"
end

file "/etc/god/master.conf" do
  owner    "root"
  group    "root"
  mode     "0644"
  notifies :restart, "service[god]"

  home     = node["practicingruby"]["deploy"]["home_dir"] 
  god_file = "#{home}/current/config/delayed_job.god"

  content "God.load('#{god_file}') if File.file?('#{god_file}')"
end
```

A master configuration file is typically used with God to load
all of the process-specific configuration files for a whole system 
when God starts up. In our case, we only have one process to watch, 
so our master configuration is a simple one-line shim that points at the
[delayed_job.god][pr-web-dj] file that is deployed alongside our Rails 
application.

Because our `/etc/god/master.conf` file is so trivial, we directly specify 
its contents in the recipe itself rather than using one of Chef's more
complicated mechanisms for dealing with configuration files. In this
particular case, manually creating the file would certainly involve
less work, but we'd lose some of the benefits that Chef is providing here.

In particular, it's worth noticing that file permissions and ownership
are explicitly specified in the recipe, that the actual location
of the file is configurable, and that Chef will send a notification
to restart God whenever this file changes. All of these things
are the sort of minor details that are easily forgotten when
manually managing configuration files on servers.

**Running god as a system service**

God needs to be running at all times, so we want to make sure that it started on
system reboot and cleanly terminated when the system is shut down. To do that, we
can configure God to run as an Upstart service. To do that, we need to create
yet another configuration file:

```ruby
cookbook_file "/etc/init/god.conf" do
  source "god.upstart"
  owner  "root"
  group  "root"
  mode   "0644"
end
```

The `cookbook_file` command used here is similar to the `file` command, but has a
specialized purpose: To copy files from a cookbook's `files` directory to
some location on the system being automated. In this case, we're
using the `files/default/god.upstart` cookbook file as our source, and it
looks like this:

```
description "God is a monitoring framework written in Ruby"

start on runlevel [2345]
stop on runlevel [!2345]

pre-start exec god -c /etc/god/master.conf
post-stop exec god terminate
```

Here we can see exactly what commands are going to be used to start and 
shutdown God, as well as the runlevels that it will be started and
stopped on. We can also see that the `/etc/god/master.conf` file we
created earlier will be loaded by God whenever it starts up.

Now all that remains is to enable the service to run when the system
boots, and also tell it to start up right now:

```ruby
service "god" do
  provider Chef::Provider::Service::Upstart
  action   [:enable, :start]
end
```

It's worth mentioning here that if we didn't explicitly specify the
`Service::Upstart` provider, Chef would expect the service
configuration file to be written as a [System-V init
script][god-init], which are written at a much lower level of abstraction. There
isn't anything wrong with doing things that way, but Upstart
scripts are definitely more readable. 

By this point, we've already seen how Chef can be used to install packages,
manage configuration files, run arbitrary shell commands, 
and set up system services. That knowledge alone will take you far,
but let's look at one more recipe to discover a few more 
advanced features before we wrap things up.

## A recipe for setting up an Nginx web server 

The recipe we use for configuring Nginx is the most complicated one in
Practicing Ruby's cookbook, but it mostly just combines and expands upon the
concepts we've already discussed. Try to see what you can
understand of it before reading the explanations that follow, but don't
worry if every last detail isn't immediately clear to you:

```ruby
node.set["nginx"]["worker_processes"]     = 4
node.set["nginx"]["worker_connections"]   = 768
node.set["nginx"]["default_site_enabled"] = false

include_recipe "nginx::default"

ssl_dir = ::File.join(node["nginx"]["dir"], "ssl")
directory ssl_dir do
  owner "root"
  group "root"
  mode  "0600"
end

domain_name = node["practicingruby"]["rails"]["host"]
bash "generate-ssl-files" do
  cwd   ssl_dir
  flags "-e"
  code <<-EOS
    DOM=#{domain_name}
    openssl genrsa -out $DOM.key 4096
    openssl req -new -batch -subj "/CN=$DOM" -key $DOM.key -out $DOM.csr
    openssl x509 -req -days 365 -in $DOM.csr -signkey $DOM.key -out $DOM.crt
    rm $DOM.csr
  EOS
  notifies :reload, "service[nginx]"
  not_if   { ::File.exists?(::File.join(ssl_dir, domain_name + ".crt")) }
end

template "#{node["nginx"]["dir"]}/sites-available/practicingruby" do
  source "nginx_site.erb"
  owner  "root"
  group  "root"
  mode   "0644"
  variables(:domain_name => domain_name)
end

nginx_site "practicingruby" do
  enable true
end
```

When you put all the pieces together, this recipe is responsible for the
following tasks:

1. Overriding some default Nginx configuration values.
2. Installing Nginx and managing it as a service.
3. Generating a self-signed SSL certificate based on a configurable domain name.
4. Using a template to generate a site-specific configuration file.
5. Enabling Nginx to serve up our Rails application.

In this recipe even more than the others we've looked at, a lot of the details
are handled behind the scenes. Let's dig a bit deeper to see what's really
going on.

**Installing and configuring Nginx**

We rely on the nginx cookbook to do most of the hard work of 
setting up our web server for us. Apart
from overriding a few default attributes, we only need to include the
`nginx:default` recipe into our own code to install the relevant software 
packages, generate an `nginx.conf` file, and to provide all the necessary
init scripts to manage Nginx as a service. The following four lines
of code take care of all of that for us:

```ruby
node.set["nginx"]["worker_processes"]     = 4
node.set["nginx"]["worker_connections"]   = 768
node.set["nginx"]["default_site_enabled"] = false

include_recipe "nginx::default"
```

The interesting thing to notice here is that unlike the typical server
configuration file, only the things we explicitly changed are visible here.
All the rest of the defaults are set automatically for us, and we don't
need to be concerned with their values until the time comes when we decide we
need to change them. By hiding all the details that do not matter to us,
Chef recipes tend to be much more intention revealing than
the typical server configuration file.

**Generating SSL keys**

In a real production environment, we would probably copy SSL credentials
into place rather than generating them on the fly. However, since
this particular cookbook provides a blueprint for building an experimental testbed
rather than an exact clone of our live system, we handle this task internally to make the system a little bit more developer-friendly.

The basic idea behind the following code is that we want to generate an SSL
certificate and private key for whatever domain name you'd like, so that 
it is possible to serve up the application over SSL within a virtualized 
staging environment. But since that is somewhat of an obscure use case, you
can focus on what interesting Chef features are being used
in the following code rather than the particular shell code being executed:

```ruby
ssl_dir = ::File.join(node["nginx"]["dir"], "ssl")
directory ssl_dir do
  owner "root"
  group "root"
  mode  "0600"
end

domain_name = node["practicingruby"]["rails"]["host"]
bash "generate-ssl-files" do
  cwd   ssl_dir
  flags "-e"
  code <<-EOS
    DOM=#{domain_name}
    openssl genrsa -out $DOM.key 4096
    openssl req -new -batch -subj "/CN=$DOM" -key $DOM.key -out $DOM.csr
    openssl x509 -req -days 365 -in $DOM.csr -signkey $DOM.key -out $DOM.crt
    rm $DOM.csr
  EOS
  notifies :reload, "service[nginx]"
  not_if   { ::File.exists?(::File.join(ssl_dir, domain_name + ".crt")) }
end
```

As you read through this code, you may have noticed that `::File` is used
instead of `File`, which looks a bit awkward. The problem here is that
Chef defines its own `File` class that ends up having a naming collision with
Ruby's core class. So to safely make use of Ruby's `File` class, we need to
explicitly do our constant lookup from the top-level namespace. This is just a
small side effect of how Chef's recipe DSL is implemented, but it is
worth noting to clear up any confusion.

With that distraction out of the way, we can skip right over the `directory`
code which we've seen in earlier recipes, and turn our attention to the `bash`
command and its options. This example is far more interesting than the one we
used to update RubyGems earlier, because in addition to specifying a command to
execute and a `not_if` guard clause, it also does all of the following things:

* Switches the working directory to the SSL directory we created within our Nginx directory.
* Sets the `-e` flag, which will abort the script if any command fails to run successfully.
* Uses a service notification to tell Nginx to reload its configuration files

From this we see that executing shell code via a Chef recipe isn't quite the
same thing as simply running some commands in a console. The entire surrounding
context is also specified and verified, making it a whole lot more likely
that things will work the way you expect them to. If these benefits were
harder to see in the Ruby installation recipe, they should be easier to
recognize now.

**Configuring Nginx to serve up Practicing Ruby**

Although the [nginx cookbook](https://github.com/opscode-cookbooks/nginx) takes care 
of setting up our `nginx.conf` file for us, it does not manage site 
configurations for us. We need to take care of that ourselves and
tweak some settings dynamically, so that means telling our
recipe to make use of a template:

```ruby
template "#{node["nginx"]["dir"]}/sites-available/practicingruby" do
  source "nginx_site.erb"
  owner  "root"
  group  "root"
  mode   "0644"
  variables(:domain_name => domain_name)
end
```

The [full template](https://github.com/elm-city-craftworks/practicing-ruby-cookbook/blob/master/templates/default/nginx_site.erb)
is a rather long file full of the typical Nginx boilerplate, but the small
excerpt below shows how it is customized using ERB to insert some dynamic
content:

```erb
server {
  listen 80;
  server_name <%= "#{@domain_name} www.#{@domain_name}" %>;
  rewrite ^ https://$server_name$request_uri? permanent;
}
```

Once the configuration file is generated and stored in the right place, we
enable it using the following command:

```ruby
nginx_site "practicingruby" do
  enable true
end
```

Under the hood, the [nxensite](https://github.com/Dreyer/nxensite) script is used 
to do the actual work of enabling the site, but that implementation detail is 
deliberately kept hidden from view.

At this point, we have studied enough features of Chef to establish a basic
literacy that will facilitate reading a wide range of recipes with only
a little bit of effort. At the very least, you now have enough
knowledge to make sense of every recipe in Practicing Ruby's cookbook.

## A cookbook for building a (mostly) complete Rails environment 

The goal of this article was to give you a sense of what kinds of building
blocks that Chef recipes are made up of so that you could see various
infrastructure automation concepts in practice. If you feel like you've
made it that far, you may now be interested in looking at how a complete
automation project is sewn together.

The full [Practicing Ruby cookbook][pr-cookbook] contains a total of eight recipes,
three of which we've already covered in this article. The five recipes
we did not discuss are responsible for handling the
following chores:

* Creating and managing a deployment user account to be used by Capistrano.
* Installing PostgreSQL and configuring a database for use with our Rails app.
* Configuring Unicorn and managing it as an Upstart service.
* Setting up some folders and files needed to deploy our Rails app.
* Installing and managing MailCatcher as a service, to make email testing easier.

If you are curious about how these recipes work, go ahead and read them! Many
are thin wrappers around external cookbook dependencies, and none of them use
any Chef features that we haven't already discussed. Attempting to
make sense of how these recipes work would be a great way to test your 
understanding of what we covered in this article.

If you want to take things a step farther, you can actually try to provision a
production-like environment for Practicing Ruby on your own system. The
cookbook's [README file](https://github.com/elm-city-craftworks/practicing-ruby-cookbook#readme) is fairly detailed, and we have things set up to work within a
virtual machine that can run in isolation without having a negative impact
on your own development environment. We also simplify a few things to make
setup easier, such as swapping out GitHub authentication for OmniAuth developer
mode, making most service integrations optional, and other little tweaks that
make it possible to try things out without having to do a bunch of 
configuration work.

I absolutely recommend trying to run our cookbook on your own to learn a whole
lot more about Chef, but fair warning: to do so you will need to become familiar
with the complex network of underplumbing that we intentionally avoided
discussing in this article. It's not too hard to work your way through, but
expect some turbulence along the way.

## Epilogue: What are the costs of infrastructure automation?

The process of learning from Practicing Ruby's cookbook, and the act
of writing this article really convinced me that I had greatly underestimated
the potential benefits that infrastructure automation has to offer. However, it
is important to be very clear on one point: there's no such thing as a 
free lunch.

At my current stage of understanding, I feel the same about Chef as I do about
Rails: impressed by its vast capabilities, convinced of its utility, and shocked
by its complexity. There are a tremendous amount of moving parts that you need
to understand before it becomes useful, and many layers of subsystems that need
to be wired up before you can actually get any of your recipes to run.

Another concern is that "infrastructure as code" comes with the drawbacks
associated with code and not just the benefits. Third-party cookbooks vary in
quality and sometimes need to be patched or hacked to get them to work the way
you want, and some abstractions are leaky and leave you doing some tedious work 
at a lower level than you'd want. Dependency management is also complicated: using external cookbooks means introducing at least one more fragile package 
installer into your life.

In the case of Chef in particular, it is also a bit strange that although its
interface is mostly ordinary Ruby code, it has developed in a somewhat parallel
universe where the user is assumed to know a lot about system administration,
and very little about Ruby. This leads to some design choices that aren't
necessarily bad, but are at least surprising to an experienced Ruby developer.

And as for infrastructure automation as a whole, well... it doesn't fully free
you from knowing quite a few details about the systems you are trying to manage.
It does allow you to express ideas at a higher level, but you still need to
be able to peel back the veneer and dive into some low level system
administration concepts whenever something doesn't work the way you expect it
would or doesn't support the feature you want to use via its high level
interface. In that sense, an automated system will not necessarily reduce
learning costs, it just has you doing a different kind of learning.

Despite all these concerns, I have to say that this is one skillset that I wish
I had picked up years ago, and I fully intend to look for opportunities
to apply these ideas in my own projects. I hope after reading this article,
you will try to do the same, and then share your stories about your experiences.

## Recommendations for further reading

Despite having a very complex ecosystem, the infrastructure automation world
(and especially the Chef community) have a ton of useful documentation that is
freely available and easy to get started with. Here are a few resources to try
out if you want to continue exploring this topic on your own:

* [Opscode Chef documentation](http://docs.opscode.com): The official Chef documentation; comprehensive and really well organized. 

* [Opscode public cookbooks](https://github.com/opscode-cookbooks): You can learn a lot by reading some of the most widely-used cookbooks in the Chef community. For complex examples, definitely check out the [apache2](https://github.com/opscode-cookbooks/apache2) and [mysql](https://github.com/opscode-cookbooks/mysql) cookbooks.

* [#learnchef](https://learnchef.opscode.com/): A collection of tutorials and screencasts designed to help you learn Chef.

* [Common Idioms in Chef Recipes](http://www.opscode.com/blog/2013/09/04/demystifying-common-idioms-in-chef-recipes/): Explanation of (possibly surprising) idioms that sometimes appear in recipe code.

* [Learning Chef](http://mlafeldt.github.io/blog/2012/09/learning-chef): A friendly introduction to Chef written by Mathias.

If you've got some experience with infrastructure automation and have found
other tutorials or articles that you like which aren't listed here, please leave
a comment. Mathias will also be watching the comments for this article, so
don't be afraid to ask any general questions you have about infrastructure
automation or Chef, too.

Thanks for making it all the way to the end of this article, and happy automating!

[puppet]: http://projects.puppetlabs.com/projects/puppet
[chef]: http://www.opscode.com/chef/
[pr-cookbook]: https://github.com/elm-city-craftworks/practicing-ruby-cookbook/tree/1.0.8
[pr-cookbook-attributes]: https://github.com/elm-city-craftworks/practicing-ruby-cookbook/blob/1.0.8/attributes/default.rb
[pr-web]: https://github.com/elm-city-craftworks/practicing-ruby-web
[chef-attributes]: http://docs.opscode.com/essentials_cookbook_attribute_files.html
[God]: http://godrb.com/
[god-init]: https://raw.github.com/elm-city-craftworks/practicing-ruby-cookbook/37ca12dc6432dfee955a70b6f2cc288e40782733/files/default/god.sh
[pr-web-dj]: https://github.com/elm-city-craftworks/practicing-ruby-web/blob/master/config/delayed_job.god

