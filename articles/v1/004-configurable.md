In [Issue #3](http://practicingruby.com/articles/31), we looked at the downsides
of mixing configuration code with application code. We discussed how storing
configuration data in YAML files can solve many of those issues, but not
all of them. In this article, we will explore the limitations of the YAML 
format, and then consider the tradeoffs involved in using various 
alternative solutions.

### Dynamic Configuration

In response to the questions posed by Issue #3, Franklin Webber demonstrated
how YAML's aliasing functionality can be used to reduce duplication in
a configuration file:

```
default: &DEFAULT
  host:
    name: testsystem
    http_port: '8080'
    username: defaultuser
  database:
    host: db01/db01
    username:
    password:
  test:
    browser: FIREFOX

windows_default: &WIN_DEFAULT
  <<: *DEFAULT
  test:
    browser: IE
```

In this example, the `default` and `windows_default` configurations share almost
the same attributes, except that browsers differ in test mode. Franklin uses
aliasing to merge the `DEFAULT` data into the `WIN_DEFAULT` entry, solving his
duplication problem. This is a neat way to keep your YAML configurations well
organized.

While Franklin shared this example of aliasing to illustrate that some dynamic
functionality does exist within YAML, he acknowledged that the format was still
mostly suited for static data. Even though it is possible to reference
various entries within the data structure, they cannot be manipulated. 
That means that the following concatenation example cannot be done in pure 
YAML, and would require some additional processing:

```
host:
  name: localhost
  port: 3000
web:
  login_url: #{name}:#{port}/login 
```

This is where we cross the line from problems solved by a data format to those
solved by programming languages. Franklin suggests that running the YAML data
through Ruby's `eval` method is an option, which is similar to how Rails
passes its YAML files through `ERB`. This approach would work, but once we 
start going down that road, we need to ask what it would take to implement 
the entire configuration in pure Ruby. As you can see in the following example, 
the answer is 'not much':

```ruby
module MyApp
  module Config
    HOST = { :name => 'localhost', :port => 3000 }
    WEB  = { :login_url =>  "#{HOST[:name]}:#{HOST[:port]}/login" }
  end
end
```

If we drop this snippet into our application code, we run into the same problems
that we saw in the first example in Issue #3. But by defining this module
in its own file and requiring that file, those issues are avoided:

```ruby
require "config/my_app_config"
require "rest_client"

module MyApp
  module Client
    extend self

    def authenticate(user, password)
      RestClient.post(MyApp::Config::WEB[:login_url], 
        :user => user, :password => password)
    end
  end
end

MyApp::Client.authenticate('my_user', 'seekrit')
```

Using ordinary Ruby constants is no more complicated than referring to data
stored in a YAML file, but gives you the full power of Ruby in your
configuration scripts. In more complex configurations, you may even build
a mini-DSL, as shown in the following example:

```ruby
AccessControl.configure do
  role "basic", 
    :permissions => [:read_answers, :answer_questions]
  
  role "premium", 
    :parent      => "basic",
    :permissions => [:hide_advertisements]

  role "manager", 
    :parent      => "premium",
    :permissions => [:create_quizzes, :edit_quizzes]

  role "owner",
    :parent      => "manager",
    :permissions => [:edit_users, :deactivate_users]
end
```

While this looks like vanilla configuration code on the surface, we can see that what we're working with are full blown Ruby objects. Here are some examples of how this system is used:

```ruby
>> AccessControl.roles_with_permission(:create_quizzes)
=> ["manager", "owner"]
>> AccessControl["premium"].permissions
=> [:hide_advertisements, :read_answers, :answer_questions]
>> AccessControl["owner"].allows?(:edit_users)
=> true
>> AccessControl["basic"].allows?(:edit_users)
=> false
```

This is an advanced configuration system that not only encapsulates some configuration data, but also makes it possible to query that data in useful ways. The following implementation code illustrates how little magic is involved in building such a system.

```ruby
module AccessControl
  extend self
 
  def configure(&block)
    instance_eval(&block)
  end

  def definitions
    @definitions ||= Hash.new
  end

  def role(level, options={})  
    definitions[level] = Role.new(level, options)
  end

  def roles_with_permission(permission)
    definitions.select { |k,v| v.allows?(permission) }.map { |k,_| k }
  end

  def [](level)
    definitions[level]
  end

  class Role
    def initialize(name, options)
      @name        = name
      @permissions = options[:permissions]
      @parent      = options[:parent]
    end

    attr_reader :parent

    def permissions
      return @permissions unless parent
      
      @permissions + AccessControl[parent].permissions
    end

    def allows?(permission)
      permissions.include?(permission)
    end
    
    def to_s
      @name
    end
  end
end
```

Because doing configuration in pure Ruby is so easy, I often lean towards it rather than using YAML or some other external file format. I find configuration files written in Ruby to be just as readable as YAML, but far more flexible.

There are some situations in which external data formats make more sense than Ruby based configurations. Using YAML might be a better idea than the approach shown above if any of the following apply to your application:

 * You need to integrate with other programs that will either read or write your configuration files. It is easier for a program written in another language to produce and consume YAML than it is for it to work with arbitrary Ruby code

 * You don't want users to be able to execute arbitrary code in your application's runtime environment. This can either be for security reasons, or for protecting users from their own stupidity by restricting the range of possible mistakes they can make.

 * You want configuration data that can easily be passed over a network and then executed remotely.

While these are all good reasons to avoid Ruby based configurations, frankly they are not common scenarios. The reason Ruby has had such a widespread adoption of YAML is almost certainly not because of it being the best tool for the job, but instead due to an early design decision made in Rails that people have emulated in their own projects without further thought. While either technique may get the job done, I'd argue that Ruby based configurations are a better default choice due to their inherent flexibility.

But sometimes, neither Ruby nor YAML does what we need them to do. In certain situations, configuration data isn't made available until the application is invoked. For those scenarios, we can take advantage of how well Ruby is integrated with the shell by making use of environment variables.

### Using the Shell Environment for Configuration

Every Ruby application has a fairly primitive but useful configuration system built into it through direct access to shell environment variables. As you can see in the code below, Ruby provides a top level constant that turns the environment variable mappings into a plain old Hash object.

```
$ TURBINE_API_KEY="saf3t33553" ruby -e "puts ENV['TURBINE_API_KEY']"
IqxPfasfasasfasfgqNm
```

The fact that I mention API keys in the above contrived example is no coincidence. The area I first made use of environment variables in my own applications was in a command line application which acted as a client to a web service I needed to interact with. Each distinct user needed to use a different API key, but I didn't want to rely on fragile home directory lookup code to provide per-user configuration. By using environment variables, it was possible to write a line like the following in my <i>.bash_profile</i> which would ensure that this information was available whenever my command line program ran.

```
export TURBINE_API_KEY="IqxPfasfasasfasfgqNm"
```

Since most modern shell implementations support environment variables, they're a good choice for this sort of semi-global configuration data. You'll also find environment variables used in places where you don't have much control over the system where your application is destined to run. The Ruby web application deployment service Heroku is a good example of that sort of environment.

On Heroku, you aren't given direct shell access and aren't even given any guarantees about where on the filesystem your application is destined to run. On top of that, if you want to run an open source application on Heroku while actively mirroring your changes to Github or some other public git host, you can't simply check in configuration files which may contain sensitive information, whether written in Ruby, YAML, or anything else.

The way Heroku solves these problems is with a configuration system based on, you guessed it, environment variables. The following example from the Heroku website shows how these set via the heroku command line app.

```
$ cd myapp
$ heroku config:add S3_KEY=8N029N81 S3_SECRET=9s83109d3+583493190
Adding config vars:
  S3_KEY    => 8N029N81
  S3_SECRET => 9s83109d3+583493190
Restarting app...done.
```

In the application, these variables are accessed in a similar fashion to our
previous example:

```ruby
AWS::S3::Base.establish_connection!(
  :access_key_id     => ENV['S3_KEY'],
  :secret_access_key => ENV['S3_SECRET']
)
```

While hardly the first tool you should reach for, environment variables make sense in situations in which you do not want to store sensitive information within your application. They also come in handy when you don't want to assume anything about your user's file system in order to locate user-wide configuration settings.

Before we wrap up with some general tips that are relevant to all configurable applications, I'd like to quickly visit one more trick that involves project-wide configurations.

### Per-project configurations for command line apps

Some command line applications need to be context aware in order to do their jobs. Two such examples are rake and git. Both tools know how to locate their own configuration information so that they do the right thing when running their commands.

For example, git knows which repository to interact with because it knows how to work backwards to find the <i>.git/</i> configuration folder at the project root. Likewise, running `rake test` from anywhere within your project causes rake to look backwards recursively until it finds the nearest <i>Rakefile</i> to run. This general pattern can be seen in many other applications, and is worth knowing about in case you ever need to make use of it yourself.

While I don't want to go into much detail about this topic, I will say that it seemed a bit magical to me until I needed to implement this sort of functionality in my own projects. The basic idea is no more complicated than working backwards from your current directory until you find the file or folder than you need to interact with, which is something Ruby's pathname library can make quick work of.

Here's an example pulled directly out of a project of mine which illustrates a reverse search from the current working directory back to the filesystem's root directory.

```ruby
require 'pathname'

def config_dir(dir = Pathname.new("."))
  app_config_dir = dir + ".myappconfigfolder"
  if dir.children.include?(app_config_dir)
    app_config_dir.expand_path
  else
    return nil if dir.expand_path.root?
    config_dir(dir.parent)
  end
end
```

A bit of code like this combined with ordinary `require` calls for Ruby configurations or `YAML.load_file` calls for YAML configurations can be used to implement exactly the sort of context sensitive behavior you find in rake and git. I'll leave the exact methods of doing that as something for you to explore on your own, but hopefully this bit of code will come in handy if you ever run into that sort of situation.

This article turned out to be longer than I expected it to be, but hopefully was still quite useful to you. Before we part, let's review a few key points to keep in mind when building any sort of configuration system.

### Configuration Best Practices 

* Convention often is better than configuration. Always provide sensible defaults where possible. For example, if you're interacting with a service that has a common default port, don't force the user to define a port to use unless they wish to deviate from the default.

* Don't put your real configuration files into your application's code repository, since this can expose sensitive data and also makes it hard for others to submit patches without merge conflicts on configuration settings.

* Include a sample configuration file filled with reasonable defaults with your application. For example, in Rails, people often check in a <i>config/database.yml.example</i> for this purpose. The goal should be to make it as easy for your user to make a copy of the sample file and then customize it as needed to get their systems up and running

* Raise an appropriate error message when a config file is missing. You can do this by doing a `File.exist?` check before loading your configuration file, or by rescuing the error a failed load causes and then re-raising a more specific error that instructs the user on where to set up their configuration file.

* Make it very easy for users to override defaults by merging their overrides rather than forcing them to replace whole configuration structures in order to make a small change.

### Reflections 

What do you think of what we've covered here? Feel free to leave your questions, comments and suggestions in the comments section below.
  
> **NOTE:** This article has also been published on the Ruby Best Practices blog. There [may be additional commentary](http://blog.rubybestpractices.com/posts/gregory/033-issue-4-configurable.html#disqus_thread) 
over there worth taking a look at.
