Ruby developers tend to prefer convention over configuration, but that doesn't mean our applications are configuration-free.  If you're doing serious software development, it's likely that at least some of your projects depend on some sort of configuration data. Whether you simply need to store database credentials, an API key, or something much more complicated, it's important to know how to do so in a way that is flexible without introducing too much administrative overhead.

In this two part article series, we'll be talking about the many options Ruby provides us for working with configuration data, and what techniques work best in various common scenarios. We'll start by showing a single example of a problem and one way to solve it, and then go on to discuss various other options in Issue #4.

### Configuration Done Wrong

The worst way to work with configuration data is to embed it directly within your application. The simple Sinatra application shown below is a nice example of what *not* to do.

```ruby
require "rubygems"
require "sinatra"
require "active_record"

class User < ActiveRecord::Base; end

configure do
  ActiveRecord::Base.establish_connection(
    :adapter  => "mysql",
    :host     => "myhost",
    :username => "myuser",
    :password => "mypass",
    :database => "somedatabase"
  )
end

get "/users" do
  @users = User.all
  haml :user_index
end
```

The code above establishes a connection to the database on application startup and then proceeds to implement a rather simple call to get a full user listing and then render a Haml template. With an application this simple, the configuration data seems a bit harmless. But with just a moment's thought, it is easy to see numerous flaws with this sort of design.

The first and most obvious issue with this sort of code is security, everyone who looks at its source needs to be trusted, as the credentials for the database connection are embedded directly within it. Now, this may or may not be a concern depending on who is involved with the project, and what other systems are in place to restrict access to production systems, but it is important to think about nonetheless.

In a field in which revision control is a key part of our practices, it's not as simple as removing this sensitive information when you decide you no longer want to share it with others. Rewriting the history of a repository is straightforward on its own, but mixing application and configuration code makes it tricky to do this without jumping through a bunch of hoops. This is where the security concerns overlap with maintenance issues.

Suppose you want to share this trivial sinatra application with a friend, or even use it on another machine. The in-application configuration forces everyone to set up an identical database environment, even if the needs of the application may not really call for that. Any change to this configuration information would lead to merge conflicts when you try to pull in changes across machines, which could become annoying quite fast.

Fortunately, Ruby makes writing proper configuration systems easy enough where the only valid reason for writing code this way is if you're doing a throwaway spike. Let's see how easily we can emulate the way Rails solves this problem in their own framework.

### YAML Based Configurations

With slight modifications, we can move our configuration out of our application and into a YAML file. We'd like to end up with a database.yml file looking quite similar to a standard Rails configuration file, such as the one below:

```
development:
  adapter: mysql
  database: mydatabase
  username: myuser
  password: mypass
  host: myhost
```

Through the standard YAML library, we can easily access this data by parsing it into a nested hash, as shown in the irb session below.

```
>> require "yaml"
=> true
>> YAML.load_file("config/database.yml")
=> {"development"=>{"username"=>"myuser", "adapter"=>"mysql", 
   "database"=>"mydatabase", "host"=>"myhost", "password"=>"mypass"}}
```

If we compare this output to our original example of calling `establish_connection()` directly with an explicit configuration hash, the following code should be very easy to follow.

```ruby
require "rubygems"
require "yaml"
require "sinatra"
require "active_record"

class User < ActiveRecord::Base; end

configure do
  database_config = YAML.load_file("config/database.yml")
  ActiveRecord::Base.establish_connection(database_config)
end

get "/users" do
  @users = User.all
  haml :user_index
end
```

By removing the configuration data from the application code, we have made it so that the application code no longer needs to be modified everywhere it runs, provided the configuration data is properly set up. We can now safely tell our revision control system to ignore the configuration file without it causing many problems.

Now that we've seen a simple problem and a reasonable fix for it, let's ponder a few questions so that we can hit some more subtle topics in Issue #4

### Questions and Discussion Points

* YAML is a nice readable data format with good Ruby support, but it can only represent data, which does not allow you to make dynamic configuration systems with it. Rails runs its YAML files through ERB to address this issue, but what other ways could this problem be solved?

* How would you handle configuration for something like a command line application which may be run anywhere on your system? How might you build per-user and per-project configuration systems?

* Suppose you have a project that is mirrored to both Github and Heroku, and that you want to run directly from your public sources while providing some configuration options in your production environment. How should you handle this?

* What are some important practices to follow when implementing configuration systems, regardless of the underlying context and what approach you choose?

Please feel free to include your answers to these questions in the comments section below, along with any other thoughts or questions you might wish to share. I promise to reply personally to anyone who leaves a comment!
  
> **NOTE:** This article has also been published on the Ruby Best Practices blog. There [may be additional commentary](http://blog.rubybestpractices.com/posts/gregory/032-issue-3-configurable.html#disqus_thread) 
over there worth taking a look at.
