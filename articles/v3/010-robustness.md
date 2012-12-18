Writing robust code is always challenging, even when dealing with extremely well controlled environments. But when you enter the danger zone where software failures can result in data loss or extended service interruptions, coding for robustness becomes essential even if it is inconvenient to do so. 

In this article, I will share some of the lessons I've learned about building
stable software through my work on the [Newman mail framework](https://github.com/mendicant-original/newman). While the techniques I've discovered so far are fairly ordinary, it was easy to underestimate their importance in the early stages of the project's development. My hope is that by exposing my stumbling points, it will save others from making the same mistakes.

### Lesson 1: Collect enough information about your workflow.

In many contexts, collecting the information you need to analyze a failure is the easy part of the debugging process. When your environment is well controlled, a good stack trace combined with a few well placed `puts` statements are often all you need to start reproducing an error in your development environment. Unfortunately, these well-worn strategies are not nearly as effective for debugging application frameworks.

To get a clearer sense of the problem, consider that Newman's server software knows almost nothing about the applications it runs, nor does it know much of anything about the structures of the emails it is processing. It also cannot assume that its interactions with external IMAP and SMTP servers will be perfectly stable. In this kind of environment, something can go wrong at every possible turn. This means that in order to find out where and why a failure occured, it is necessary to make the sequence of events easier to analyze by introducing some sort of logging system.

A good place to start when introducing event logging is with the outermost layer of the system. In Newman's case, this means tracking information about every incoming and outgoing email, as shown below: 

```
I, [2012-03-10T12:46:57.274091 #9841]  INFO -- REQUEST: 
{:from=>["gregory_brown@letterboxes.org"], :to=>["test+ping@test.com"],
:bcc=>nil, :subject=>"hi there!", :reply_to=>nil}

I, [2012-03-10T12:46:57.274896 #9841]  INFO -- RESPONSE: 
{:from=>["test@test.com"], :to=>["gregory_brown@letterboxes.org"], 
:bcc=>nil, :subject=>"pong", :reply_to=>nil}
```

Because Newman currently only understands how to filter messages based on their TO and SUBJECT fields, the standard log information is fairly helpful for basic application debugging needs. However, when dealing with complex problems, it is nice to be able to see [the raw contents of the messages](https://raw.github.com/gist/01fbab481a21f4d43bbf/0778e1a0ae887e6423bce985298e3f8d60eb37a0/gistfile1.txt). Rather than choosing one or the other, Newman handles both log formats by outputting them at different log levels:

```ruby
module Newman
  module EmailLogger
    def log_email(logger, prefix, email)
      logger.debug(prefix) { "\n#{email}" }
      logger.info(prefix) { email_summary(email) }
    end

    private

    def email_summary(email)
      { :from     => email.from,
        :to       => email.to,
        :bcc      => email.bcc,
        :subject  => email.subject,
        :reply_to => email.reply_to }
    end
  end
end
```

Having the ability to dynamically decide what level of detail your log output should contain is one of the main advantages of using a proper logging system instead of directly outputting messages to the console. While it would be possible to implement some sort of information filtering mechanism without using a formal logging system, doing so would involve reinventing many of the things that the `Logger` standard library already provides for you.

The cost of introducing a logging system is that once you depend on logs for your debugging information, some form of exception logging becomes absolutely necessary. Because failures can be very context-dependent, deciding how handle them can be tricky. 

### Lesson 2: Plan for various kinds of predictable failures.

Because Newman does not know anything about the applications it runs except that they all implement a `call` method, it is not possible to be selective about what kinds of errors to handle. Instead, a catch-all mechanism is implemented in the `process_request` method:

```ruby
module Newman
  class Server
    def process_request(request, response)
      apps.each do |app|
        app.call(:request  => request, 
                 :response => response, 
                 :settings => settings,
                 :logger   => logger)
      end

      return true
    rescue StandardError => e
      if settings.service.raise_exceptions
        raise
      else
        logger.info("APP ERROR")  { e.inspect }
        logger.debug("APP ERROR") { "#{e.inspect}\n" + e.backtrace.join("\n  ") }

        return false
      end
    end
  end
end
```

If you trace the execution path through this method, you'll find that there are three possible outcomes. If everything worked as expected, the method simply returns true. However, if an exception is raised by one of the applications, then the `raise_exceptions` configuration setting is used to determine whether to simply re-raise the exception or log the error and return false.

The reason `Newman::Server#process_request` is implemented in this somewhat awkward way is that it is necessary to let the application developer determine whether or not application errors should crash the server. Generally speaking, this would be a bad behavior in production, because it means that a single fault in an edge case of a single feature could halt a whole service that is otherwise working as expected. However, when it comes to writing tests, it might be nice for applications to raise their exceptions rather than quietly writing stack traces to a log file and moving on. This pair of competing concerns explains why the `raise_exceptions` configuration option exists, even if it leads to ugly implementation code.

While `Newman::Server#process_request` does a good job of handling application errors, there are a range of failures that can happen as a result of server operations as well. This means that `Newman::Server#tick` needs to implement its own exception handling and logging, as shown below:

```ruby
module Newman
  class Server
    def tick         
      mailer.messages.each do |request| 
        response = mailer.new_message(:to   => request.from, 
                                      :from => settings.service.default_sender)

        process_request(request, response) && response.deliver
      end
    rescue StandardError => e
      logger.fatal("SERVER ERROR") { "#{e.inspect}\n" + e.backtrace.join("\n  ") }
      raise
    end
  end
end
```

While it may be possible to recover from some of the errors that occur at the server level, there are many problems which simply cannot be recovered from automatically. For this reason, `Newman::Server#tick` always re-raises the exceptions it encounters after logging them as fatal errors. While implementing this method in such a conservative way helps shield against dangerous failures, it does not completely prevent them from occurring. Sadly, that is a lesson I ended up learning the hard way.

### Lesson 3: Reduce the impact of catastrophic failures. 

A few days before this article was published, I accidentally introduced an
infinite send/receive loop into the experimental Newman-based mailing list system [MailWhale](https://github.com/mendicant-original/mail_whale). I caught the problem right away, but not before my email provider banned me for 1 hour for exceeding my send quota. In the few minutes of chaos before I figured out what was going wrong, there was a window of time in which any incoming emails would simply be dropped, resulting in data loss.

It's painful to imagine what would have happened if this failure occured while someone wasn't actively babysitting the server. While the process was crashing with a `Net::SMTPFatalError` each time cron ran it, this happened after reading all incoming mail. As a result, the incoming mail would get dropped from the inbox without any response, failing silently. Once the quota was lifted, a single email would cause the server to start thrashing again, eventually leading to a permanent ban. In addition to these problems, anyone using the mailing list would be bombarded with at least a few duplicate emails before the quota kicked in each time. Although I was fortunate to not live out this scenario, the mere thought of it sends chills down my spine.

While the infinite loop I introduced could probably be avoided by doing some simple checks in Newman, the problem of the server failing repeatedly is a general defect that could cause all sorts of different problems down the line. To solve this problem, I've implemented a simplified version of the [circuit breaker](http://en.wikipedia.org/wiki/Circuit_breaker_design_pattern) pattern in MailWhale, as shown below:

```ruby
require "fileutils"

# unimportant details omitted...

begin
  if File.exists?("server.lock")
    abort("Server is locked because of an unclean shutdown. Check "+
          "the logs to see what went wrong, and delete server.lock "+
          "if the problem has been resolved") 
  end

  server.tick
rescue Exception # used to match *all* exceptions
  FileUtils.touch("server.lock")
  raise 
end
```

With this small change, any exception raised by the server will cause a lock file to be written out to disk, which will then be detected the next time the server runs. As long as the `server.lock` file is present, the server will immediately shut itself down rather than continuing on with its processing. This forces someone (or some other automated process) to intervene in order for the server to resume normal operations. As a result, repeated failure is a whole lot less likely to occur. 

If this circuit breaker were in place when I triggered the original infinite loop, I would have still exceeded my quota, but the only data loss would be the first request the server failed to respond to. All email that was sent in the interim would remain in the inbox until the problem was fixed, and there would be no chance that the server would continue to thrash without someone noticing that an unclean shutdown had occurred. This is clearly a better behavior, and perhaps this is how things should have been implemented in the first place.

Of course, we now have the problem that this code is a bit too aggressive. There are likely to be many kinds of failures which are transient in nature, and shutting down the server and hard-locking it like this feels overkill for those scenarios. However, I am gradually learning that it is better to whitelist things than blacklist them when you can't easily enumerate what can possibly go wrong. For that reason I've chosen to go with this extremely conservative solution, but I will need to put this technique through its paces a bit before I can decide whether it is really the right approach. 

### Reflections

I originally planned to cover many more lessons in this article, but the more I worked on it, the more I realized my own lack of experience in producing truly robust software. When it comes to email, it's like the entire problem space is one big fuzz test: there seems to be an infinite amount of ways for things to crash and burn.

In addition to the few issues I have already outlined, Newman is going to need to jump many more hurdles before it can be considered stable. In particular, I need to sort out the following problems:

* Sometimes connections via IMAP can hang indefinitely, so some sort of timeout logic needs to be introduced. To deal with this, I'm thinking of looking into the [retriable](https://github.com/kamui/retriable) gem.

* In one of my test runs of our simple ping/pong application, I ended up causing newman to reply to a Gmail mailer daemon, which caused a request/response loop to occur. Thankfully, Gmail's daemon gave up after a few tries, but if it didn't I would have ended up melting my inbox again. This means that Newman will need some way to deal with bounced emails. We've looked at some options for this, but most involve some pretty messy heuristics that make the cure look worse than the disease.

* Currently it is relatively straightforward to write automated tests to reproduce known issues, but very hard for me to come up with realistic test scenarios in a proactive way. This means that while we can shore up Newman's stability over time in this fashion, we'll always be trailing behind on the problems we haven't encountered yet. I need to look into whether there are some email-based acid tests I can run the server against.

* There is still a great deal of ugliness / fragility in the way Newman does its exception handling. The techniques I've shown in this article are meant to be considered a rough starting point, not a set of best practices. I plan to re-read Avdi Grimm's [Exceptional Ruby](http://exceptionalruby.com/) and see what ideas I can apply from it. When I first read that book I thought many of the techniques it recommended were overkill for day to day Ruby applications, but several of them may be just what Newman needs.

The bad news is that all of the above problems seem challenging enough to deal with, but they're likely to be just the first set of roadblocks on the highway to the danger zone. There are still a lot of unknown-unknowns that may get in my way. The good news is that because I can take my time while working on this project, the uncertainty of things is part of what makes this a fun problem to work on.

Have you ever had a similar experience of coding in a dangerous and poorly-defined environment before? If so, I'd love to hear your story, as well as any advice you might have for me.
