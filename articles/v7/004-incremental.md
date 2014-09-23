When you look at this photograph of highway construction, what do you see?

![](http://i.imgur.com/eej11xZ.jpg)

If your answer was "ugly urban decay", then you are absolutely right! But because this construction project is only a few miles away from my house, I can tell you a few things about it that reveal a far more interesting story:

* On the far left side of the photo, you can see the first half of a newly constructed suspension bridge. At the time this picture was taken, it was serving five lanes of northbound traffic.

* Directly next to that bridge, cars are driving southbound on what was formerly the northbound side of our old bridge, serving 3 lanes of traffic.

* Dominating the rest of the photograph is the mostly deconstructed southbound side of our old bridge, a result of several months of active work.

So with those points in mind, what you are looking at here is an *incremental improvement* to a critical traffic bottleneck along the main route between New York City and Boston. This work was accomplished with hardly any service interruptions, despite the incredibly tight constraints on the project. This is legacy systems work at the highest level, and there is much we can learn from it that applies equally well to code as it does to concrete.

## Case study: Improving one of Practicing Ruby's oldest features

Now that we've set the scene with a colorful metaphor, it is time to see how these ideas can influence the way we work on software projects. To do that, I will walk you through a major change we made to practicingruby.com that involved a fair amount of legacy coding headaches. You will definitely see some ugly code along the way, but hopefully a bit of cleverness will shine through as well.

The improvement that we will discuss is a complete overhaul of Practicing Ruby's content sharing features. Although I've encouraged our readers to share our articles openly since our earliest days, several awkward implementation details made this a confusing process:

* You couldn't just copy-paste links to articles. You needed to explictly click a share button that would generate a public share link for you.

* If you did copy-paste an internal link from the website rather than explicitly generating a share link, those who clicked on that link would be immediately asked for registration information without warning. This behavior was a side-effect of how we did authorization and not an intentional "feature", but it was super annoying to folks who encountered it.

* If you visited a public share link while logged in, you'd see the guest view rather than the subscriber view, and you'd need to click a "log in" button to see the comments, navbar, etc.

* Both internal paths and share paths were completely opaque (e.g. "articles/101" and "/articles/shared/zmkztdzucsgv"), making it hard to know what a URL pointed to without
visiting it.
 
Despite these flaws, subscribers did use Practicing Ruby's article sharing mechanism. They also made use of the feature in ways we didn't anticipate -- for example, it became the standard workaround for using Instapaper to read our content offline. As time went on, we used this feature for internal needs as well, whether it was to give away free samples, or to release old content to the public. To make a long story short, one of our most awkward features eventually also became one of the most important.

We avoided changing this system for quite a long while because we always had something else to work on that seemed more urgent. But after enough time had passed, we decided to pay down our debts. In particular, we wanted to make the following changes:

* We wanted to switch to subscriber-based share tokens rather than generating a new share token for each and every article. As long as a token was associated with an active subscriber, it could then be used to view any of our articles.

* We wanted to clean up and unify our URL scheme. Rather than having internal path like "/articles/101" and share path like "/articles/shared/zmkztdzucsgv", we would have a single path for both purposes that looked like this:

```
/articles/improving-legacy-systems?u=dc2ab0f9bb
```

* We wanted to make sure to be smart about authorization. Guests who visited a link with a valid share key would always see the "guest view" of that article, and logged in subscribers would always see the "subscriber view". If a key was invalid or missing, the guest would be explicitly told that the page was protected, rather than dropped into our registration process without warning.

* We wanted to make sure to make our links easy to share by copy-paste, whether it was from anywhere within our web interface, from the browser location bar, or even in the emails we send to subscribers. This meant making sure we put your share token pretty much anywhere you might click on an article link.

Laying out this set of requirements helped us figure out where the destination was, but we knew intuitively that the path to get there would be a long and winding road. The system we initially built for sharing articles did not take any of these concepts into account, and so we would need to find a way to shoehorn them in without breaking old behavior in any significant way. We also would need to find a way to do this *incrementally*, to avoid releasing a ton of changes to our system at once that could be difficult to debug and maintain. The rest of this article describes how we went on to do exactly that, one pull request at a time.

> **NOTE:** Throughout this article, I link to the "files changed" view of pull requests to give you a complete picture of what changed in the code, but understanding every last detail is not important. It's fine to dig deep into some pull requests while skimming or skipping others.

## Step 1: Deal with authorization failures gracefully

When we first started working on practicingruby.com, we thought it would be convenient to automatically handle Github authentication behind the scenes so that subscribers rarely needed to explicitly click a "sign in" button in order to read articles. This is a good design idea, but we only really considered the happy path while building and testing it.

Many months down the line, we realized that people would occasionally share internal links to our articles by accident, rather than explicitly generating public links. Whenever that happened, the visitor would be put through our entire registration process without warning, including:

* Approving our use of Github to authorize their account
* Going through an email confirmation process
* Getting prompted for credit card information

Most would understandably abandon this process part of the way through. In the best case scenario, our application's behavior would be seen as very confusing, though I'm sure for many it felt downright rude and unpleasant. It's a shame that such a bad experience could emerge from what was actually good intentions both on our part and on whoever shared a link to our content in the first place. Think of what a different experience it might have been if the visitor had been redirected to our landing page where they could see the following message:

![](http://i.imgur.com/kA3ePJI.png)

Although that wouldn't be quite as nice as getting free access to an article that someone wanted to share with them, it would at least avoid any confusion about what had just happened. My first attempt at introducing this kind of behavior into the system looked like what you see below:

```ruby
class ApplicationController < ApplicationController::Base
  # ...
  
  def authenticate
    return if current_authorization 
   
    flash[:notice] = 
      "That page is protected. Please sign in or sign up to continue"
      
    store_location
    redirect_to(root_path)
  end
end 
```

We deployed this code and for a few days, it seemed to be a good enough stop-gap measure for resolving this bug, even if it meant that subscribers might need to click a "sign in" button a little more often. However, I realized that it was a bit too naive of a solution when I received an email asking why it was necessary to click "sign in" in order to make the "subscribe" button work. My quick fix had broken our registration system. :cry:

Upon hearing that bad news, I immediately pulled this code out of production after writing a test that proved this problem existed on my feature branch but not in master. A few days later, I put together a quick fix that got my tests passing. My solution was to extract a helper method that decided how to handle authorization failures. The default behavior would be to redirect to the root page and display an error message as we did above, but during registrations, we would automatically initiate a Github authentication as we had done in the past:

```ruby
class ApplicationController < ApplicationController::Base
  # ...
  
  def authenticate
    return if current_authorization 
   
    store_location
    redirect_on_auth_failure
  end
  
  def redirect_on_auth_failure
    flash[:notice] = 
      "That page is protected. Please sign in or sign up to continue"
      
    redirect_to(root_path)
 end
end

class RegistrationController < ApplicationController
  # ...
  
  def redirect_on_auth_failure
    redirect_to login_path 
  end
end 
```

This code, though not especially well designed, seemed to get the job done without too much trouble. It also served as a useful reminder that I should be on the lookout for holes in the test suite, which in retrospect should have been obvious given the awkward behavior of the original code. As they say, hindsight is 20/20!


> HISTORY: Deployed 2013-07-26 and then reverted a few days later due to the registration bug mentioned above. Redeployed on 2013-08-06, then merged three days later. 
>
>[View complete diff](https://github.com/elm-city-craftworks/practicing-ruby-web/pull/145/files)

## Step 2: Add article slugs

When we first started working on practicingruby.com, we didn't put much thought to what our URLs looked like. In the first few weeks, we were rushing to get features like syntax highlighting and commenting out the door while keeping up with the publication schedule, and so we didn't have much energy to think about the minor details.

Even if it made sense at the time, this is one decision I came to regret. In particular, I  disliked the notion that the paths that subscribers saw (e.g. "/articles/101") were completely different than the ones we generated for public viewing (e.g. "/articles/shared/zmkztdzucsgv"), with no direct way to associate the two. When you add in the fact that both of these URL schemes are opaque, it definitely stood out as a poor design decision on our part.

Technically speaking, it would be possible to unify the two different schemes using subscriber tokens without worrying about the descriptiveness of the URLs, perhaps using paths like "/articles/101?u=dc20f9bb". However, since we would need to be messing around with article path generation as it was, it seemed like a good idea to make those paths much more attractive by adding slugs. The goal was to have a path like: "/articles/improving-legacy-systems?u=dc2ab0f9bb". 

Because we knew article slugs would be easy to implement, we decided to build and ship them before moving on to the more complicated changes we had planned to make. The pair of methods below are the most interesting implementation details from this changeset:

```ruby
class Article < ActiveRecord::Base
  # ...

  def self.[](key)
    find_by_slug(key) || find_by_id(key)
  end

  def to_param
    if slug.present?
      slug
    else
      id.to_s
    end
  end
end
```

The `Article[]` method is a drop-in replacement for `Article.find` that allows lookup by slug or by id. This means that both `Article[101]` and `Article['improving-legacy-code']` are valid calls, each of them returning an `Article` object. Because we only call `Article.find()` in a few places in our codebase, it was easy to swap those calls out to use `Article[]` instead.

The `Article#to_params` method is used internally by Rails to generate paths. So wherever `article_url` or `article_path` get called with an `Article` object, this method will be called to determine what gets returned. If the article has a slug associated, it'll return something like "/articles/improving-legacy-code". If it doesn't have a slug set yet, it will return the familiar opaque database ids, i.e. "/articles/101".

There is a bit of an inconsistency in this design worth noting: I chose to override the `to_params` method, but not the `find` method on my model. However, since the former is a method that is designed to be overridden and the latter might be surprising to override, I felt somewhat comfortable with this design decision.

Although it's not worth showing the code for it, I also added a redirect to the new style URLs whenever a slug existed for an article. By doing this, I was able to effectively deprecate the old URL style without breaking existing links. While we won't ever disable lookup by database ID, this at least preserves some consistency at the surface level of the application.

> HISTORY: Deployed 2013-08-16 and then merged the next day. Adding slugs to articles was a manual process that I completed a few days after the feature shipped.
>
> [View complete diff](https://github.com/elm-city-craftworks/practicing-ruby-web/pull/155/files)

## Step 3: Add subscriber share tokens

In theory it should have been nearly trivial to implement subscriber-based share tokens. After all, we were simply generating a random string for each subscriber and then appending it to the end of article URLs as a GET parameter (e.g. "u=dc20f9bb"). In practice, there were many edge cases that would complicate our implementation.

The ideal situation would be to override the `article_path` and `article_url` methods to add the currently logged in user's share token to any article links throughout the application. However, we weren't able to find a single place within the Rails call chain where such a global override would make sense. It would easy enough to get this kind of behavior in both our views and controllers by putting the methods in a helper and then mixing that helper into our ApplicationController, but it wasn't easy to take the same approach in our tests and mailers. To make matters worse, some of the places we wanted to use these path helpers would have access to the ones rails provided by default, but would not include our overrides, and so we'd silently lose the behavior we wanted to add.

We were unable to find an elegant solution to this problem, but eventually settled on a compromise. We built a low level object for generating the URLs with subscriber tokens, as shown below:

```ruby
class ArticleLink
  include Rails.application.routes.url_helpers

  def initialize(article, params)
    self.article = article
    self.params = params
  end

  def path(token)
    article_path(article, params_with_token(token))
  end

  def url(token)
    article_url(article, params_with_token(token))
  end

  private

  attr_accessor :params, :article

  def params_with_token(token)
    {:u => token}.merge(params)
  end
end
```

Then in our `ApplicationHelper`, we added the following bits of glue code:

```ruby
module ApplicationHelper
  def article_url(article, params={})
    return super unless current_user

    ArticleLink.new(article, params).url(current_user.share_token)
  end

  def article_path(article, params={})
    return super unless current_user

    ArticleLink.new(article, params).path(current_user.share_token)
  end
end
```

Adding these simple shims made it so that we got the behavior we wanted in the ordinary use cases of `article_url` and `article_path`, which were in our controllers and views. In our mailers and tests, we opted to use the `ArticleLink` object directly, because we needed to explicitly pass in tokens in those areas anyway. Because it was impossible for us to make this code completely DRY, this convention-based design was the best we could come up with.

As part of this changeset, I modified the redirection code that I wrote when we were introducing slugs to also take tokens into account. If a subscriber visited a link that didn't include a share token, it would rewrite the URL to include their token. This was yet another attempt at introducing a bit of consistency where there previously was none.

> HISTORY: Deployed code to add tokens upon visiting an article on 2013-08-20, then did a second deploy to update the archives and library links the next day, merged on 2013-08-23.
>
> [View complete diff](https://github.com/elm-city-craftworks/practicing-ruby-web/pull/158/files)

## Step 4: Redesign and improve broadcast mailer

I use a very basic web form in our admin panel to send email announcements out to Practicing Ruby subscribers. Originally, this feature relied on sending messages in batches, which was the simple thing to do when we assumed we'd be sending an identical message to everyone:

```ruby
class BroadcastMailer < ActionMailer::Base
  def deliver_broadcast(message={})
    @body = message[:body]

    user_batches(message) do |users|
      mail(
        :to => "gregory@practicingruby.com",
        :bcc => users,
        :subject => message[:subject]
      ).deliver
    end
  end

  private

  def user_batches(message)
    yield(message[:to]) && return if message[:commit] == "Test"

    User.where(:notify_updates => true).to_notify.
      find_in_batches(:batch_size => 25) do |group|
        yield group.map(&:contact_email)
    end
  end
end
```

Despite being a bit of a hack, this code served us well enough for a fairly long time. It even supported a basic "test mode" that allowed me to send a broadcast email to myself before sending it out everyone. However, the design would need to change greatly if we wanted to include share tokens in the article links we emailed to subscribers. We'd need to send out individual emails rather than sending batched messages, and we'd also need to implement some sort of basic mail merge functionality to handle article link generation.

I don't want to get too bogged down in details here, but this changeset turned out to be far more complicated than I expected. For starters, the way we were using `ActionMailer` in our original code was incorrect, and we were relying on undefined behavior without realizing it. Because the `BroadcastMailer` had been working fine for us in production and its (admittedly mediocre) tests were passing, we didn't notice the problem until we attempted to change its behavior. After attempting to introduce code that looked like this, I started to get all sorts of confusing test failures:

```ruby
class BroadcastMailer < ActionMailer::Base
  # NOTE: this is an approximation, but it captures the basic idea...
  def deliver_broadcast(message={})
    @body = message[:body]

    User.where(:notify_updates => true).to_notify.each do |user|
      mail(:to => user.contact_email, :subject => message[:subject]).deliver
    end
  end
end
```

Even though this code appeared to work as expected in development (sending individual emails to each recipient), in my tests, `ActionMailer::Base.deliveries` was returning N copies of the first email sent in this loop. After some more playing around with ActionMailer and semi-fruitless internet searches, I concluded that this was because we weren't using the mailers in the officially sanctioned way. We'd need to change our code so that the mailer returned a `Mail` object, rather than handling the delivery for us.

Because I didn't want that logic to trickle up into the controller, and because I expected things might get more complicated as we kept adding more features to this object, I decided to introduce an intermediate service object to handle some of the work for us, and then greatly simplify the mailer object. I also wanted to make the distinction between sending a test message and sending a message to everyone more explicit, so I took the opportunity to do that as well. The resulting code ended up looking something similar to what you see below:

```ruby
class Broadcaster
  def self.notify_subscribers(params)
    BroadcastMailer.recipients.each do |email|
      BroadcastMailer.broadcast(params, email).deliver
    end
  end

  def self.notify_testers(params)
    BroadcastMailer.broadcast(params, params[:to]).deliver
  end
end

class BroadcastMailer < ActionMailer::Base
  def self.recipients
    User.where(:notify_updates => true).to_notify.map(&:contact_email)
  end

  def broadcast(message, email)
    mail(:to => email,
         :subject => message[:subject])
  end
end
```

With this code in place, I had successfully converted the batch email delivery to individual emails. It was time to move on to adding a bit of code that would give me mail-merge functionality. I decided to use Mustache for this purpose, which would allow me to write emails that look like this:

```
Here is an awesome article I wrote:

{{#article}}improving-legacy-systems{{/article}}
```

Mustache would then run some code behind the scenes and turn that message body into the following output:

```
Here is an awesome article I wrote:

http://practicingruby.com/articles/improving-legacy-systems?u=dc20f9bb
```

As a proof of concept, I wrote a bit of code that handled the article link expansion, but didn't handle share tokens yet. It only took two extra lines in `BroadcastMailer#broadcast` to add this support:

```ruby
class BroadcastMailer < ActionMailer::Base
  # ...
  
  def broadcast(message, email)
    article_finder = ->(e) { article_url(Article[e]) }

    @body = Mustache.render(message[:body], :article => article_finder)

    mail(:to => email,
         :subject => message[:subject])
  end
end
```

I deployed this code in production and sent myself a couple test emails, verifying that the article links were getting expanded as I expected them to. I had planned to work on adding the user tokens immediately after running those live tests, but at that moment realized that I had overlooked an important issue related to performance.

Previous to this changeset, the `BroadcastMailer` was responsible for sending about 16 emails at a time (25 people per email). But now, it would be sending about 400 of them! Even though we use a DelayedJob worker to handle the actual delivery of the messages, it might take some significant amount of time to insert 400 custom-generated emails into the queue. Rather than investigating that problem right away, I decided to get myself some rest and tackle it the next day with Jordan.    

> HISTORY: Deployed on 2013-08-22, and then merged the next day.
>
> [View complete diff](https://github.com/elm-city-craftworks/practicing-ruby-web/pull/162/files)

## Step 5: Test broadcast mailer's performance

Before we could go any farther with our work on the broadcast mailer, we needed to check the performance implications of switching to non-batched emails. We didn't need to do a very scientific test -- we just needed to see how severe the slowdown was. Because our previous code ran without a noticeable delay, pretty much anything longer than a second or two would be concerning to us.

To conduct our test, we first populated our development environment with 2000 users (about 5x as many active users as we had on Practicing Ruby at the time). Then, we posted a realistic email in the broadcast mailer form, and kept an eye on the messages that were getting queued up via the Rails console. After several seconds we hadn't even queued up 100 jobs, so it became clear that performance very well could be a concern.

To double check our estimates, and to form a more realistic test, we temporarily disabled our DelayedJob worker on the server and then ran the broadcast mailer in our live environment. Although the mailer did finish up queuing its messages without the request timing out, it took about half a minute to do so. With this information in hand, we cleared out the test jobs so that they wouldn't actually be delivered, and then spent a bit of time lost in thought.

Ultimately, we learned several important things from this little experiment:

1. The mail building and queuing process was definitely slow enough to worry us.
2. In the worst case scenario, I would be able to deal with a 30 second delay in delivering broadcasts, but we would need to fix this problem if we wanted to unbatch other emails of ours, such as comment notifications.
3. The most straightforward way to deal with this problem would be to run the entire mail building and queuing process in the background.

The first two points were not especially surprising to us, but the third concerned us a bit. While we have had good luck using DelayedJob in conjunction with the MailHopper gem to send email, we had some problems in the past with trying to handle arbitrary jobs with it. We suspected this had to do with some of our dependencies being outdated, but never had time to investigate properly. With our fingers crossed, we decided to hope for the best and plan for the worst.

## Step 6: Process broadcast mails using DelayedJob

Our first stab at backgrounding the work done by
`Broadcaster.notify_subscribers`  was to simply change the call to
`Broadcaster.delay.notify_subscribers`. 

In theory, this small change should have done the
trick: the method is conceptually nothing more than a "fire and forget"
function that did not need to interact in any way with its caller. But after
spending a long time staring at an incredibly confusing error log, we
realized that it wasn't safe to assume that DelayedJob would cleanly serialize
a Rails `params` hash. Constructing our own hash to pass into the
`Broadcaster.notify_subscribers` method resolved those issues, and we ended up
with the following code in `BroadcastsController`:

```ruby
module Admin
  class BroadcastsController < ApplicationController
    def create
      # ...

      # build our own hash to avoid DelayedJob serialization issues
      message = { :subject => params[:subject],
                  :body    => params[:body] } 

      if params[:commit] == "Test"
        message[:to] = params[:to]

        Broadcaster.notify_testers(message)
      else
        Broadcaster.delay.notify_subscribers(message)
      end

      # ...
    end
  end
end
```

After tweaking our test suite slightly to take this change into account, we
were back to green fairly quickly. We experimented with the delayed broadcasts
locally and found that it resolved our slowness issue in the UI. The worker
would still take a little while to build all those mails and get them queued
up, but since it was being done in the background it no longer was much of a
concern to us.

We were cautiously optimistic that this small change might fix our issues, so
we deployed the code to production and did another live test. Unfortunately,
this lead us to a new error condition, and so we had to go back to the drawing board. 
Eventually we came across [this Github issue](https://github.com/collectiveidea/delayed_job/issues/350), which hinted (indirectly) that we might be running into one of the many issues with YAML parsing on Ruby 1.9.2.

We could have attempted to do yet another workaround to avoid updating our
Ruby version, but we knew that this was not the first, second, or even third time that we had been bitten by the fact that we were still running an ancient
and poorly supported version of Ruby. In fact, we realized that wiping the
slate clean and provisioning a whole new VPS might be the way to go, because
that way we could upgrade all of our platform dependencies at once.

So with that in mind, Jordan went off to work on getting us a new production
environment set up, and we temporarily put this particular changeset on hold.
There was still plenty of work for me to do that didn't rely on upgrading our
production environment, so I kept working against our old server while he tried to spin up a new one.

> HISTORY: Deployed for live testing on 2013-08-23 but then immediately pulled
> from production upon failure. Redeployed to our new server on 2013-08-30,
> then merged the following day.
>
> [View complete diff](https://github.com/elm-city-craftworks/practicing-ruby-web/pull/164)

## Step 7: Support share tokens in broadcast mailer

Now that we had investigated the performance issues with the mailer and had a plan in place to fix them, it was time for me to finish what I had planned to work on in the first place: adding share tokens to article links in emails.

The changes to `BroadcastMailer` were fairly straightforward: pass a `User` rather than an email address into the `broadcast` method, and then use `ArticleLink` to generate a customized link based on the subscriber's share token: 

```ruby
class BroadcastMailer < ActionMailer::Base
  def self.recipients
    User.where(:notify_updates => true).to_notify
  end

  def broadcast(message, subscriber)
    article_finder = ->(e) { 
      ArticleLink.new(Article[e]).url(subscriber.share_token) 
    }

    @body = Mustache.render(message[:body], :article => article_finder)

    mail(:to => subscriber.contact_email,
         :subject => message[:subject])
  end
end
```

The only complication of rewiring `BroadcastMailer` this way is that it broke our test mailer functionality. Because the test mailer could send a message to any email address (whether there was an account associated with it or not), we wouldn't be able to look up a valid `User` record to pass to the `BroadcastMailer`. The code below shows my temporary solution to this API compatibility problem:

```ruby
class Broadcaster
   # ...
   
  def self.notify_testers(params)
    subscriber = Struct.new(:contact_email, :share_token)
                       .new(params[:to], "testtoken")

    BroadcastMailer.broadcast(params, subscriber).deliver
  end
end
```

Using a `Struct` object to generate an interface shim as I've done here is not the most elegant solution, but it gets the job done. A better solution would be to create a container object that could be used by both `notify_subscribers` and `notify_testers`, but I wasn't ready to make that design decision yet.

With these changes in place, I was able to do some live testing to verify that we had managed to get share tokens into our article links. Now all that remained was to add the logic that would allow these share tokens to permit guest access to articles.

> HISTORY: Deployed 2013-08-24, then merged on 2013-08-29.
>
> [View complete diff](https://github.com/elm-city-craftworks/practicing-ruby-web/pull/165)

## Step 8: Allow guest access to articles via share tokens

With all the necessary underplumbing in place, I was finally ready to model the new sharing mechanism. The end goal was to support the following behavior:

- Subscribers see the full article w. comments whenever they are logged in
- With a valid token in the URL, guests see our "shared article" view.
- Without a valid token, guests see the "protected page" error
- Links that use a token from an expired account are disabled
- Old-style share links redirect to the new-style subscriber token links

The main challenge was that there wasn't an easy way to separate these concepts from each other, at least not in a meaningful way. However, we were able to reuse large chunks of existing code to do this, so most of the work was just tedious rewiring of controller actions while layering in a few more tests here and there.

The changes that needed to be made to support these behaviors were not that hard to make, but I did feel concerned about how complicated our `ArticlesController#show` action was getting. Including the relevant filters, here is what it looked like after all the changes were made (skim it, but don't bother trying to understand it!):

```ruby
class ArticlesController < ApplicationController
  before_filter :find_article, :only => [:show, :edit, :update, :share]
  before_filter :update_url, :only => [:show]
  before_filter :validate_token, :only => [:show]

  skip_before_filter :authenticate, :only => [:show, :shared, :samples]
  skip_before_filter :authenticate_user, :only => [:show, :shared, :samples]

  def show
    store_location
    decorate_article

    if current_user
      mixpanel.track("Article Visit", :title => @article.subject,
                                      :user_id => current_user.hashed_id)

      @comments = CommentDecorator.decorate(@article.comments
                                                    .order("created_at"))
    else
      shared_by = User.find_by_share_token(params[:u]).hashed_id

      mixpanel.track("Shared Article Visit", :title => @article.subject,
                                            :shared_by => shared_by)

      render "shared"
    end
  end

  private

  def find_article
    @article = Article[params[:id]]

    render_http_error(404) unless @article
  end

  def update_url
    slug_needs_updating = @article.slug.present? && params[:id] != @article.slug
    missing_token = current_user && params[:u].blank?

    redirect_to(article_path(@article)) if slug_needs_updating || missing_token
  end

  def validate_token
    return if current_user.try(:active?)

    unless params[:u].present? && 
           User.find_by_share_token_and_status(params[:u], "active")
      attempt_user_login # helper that calls authenticate + authenticate_user
    end
  end
end
```

This is clearly not a portrait of healthy code! In fact, it looks suspiciously similiar to the code samples that "lost the plot" in Avdi Grimm's contributed article on [confident coding](https://practicingruby.com/articles/confident-ruby). That said, it's probably more fair to say that there wasn't much of a well defined plot when this code was written in the first place, and my attempts to modify it only muddied things further. 

It was hard for me to determine whether or not I should attempt to refactor this code right away or wait until later. From a purely technical perspective, the answer was obvious that this code needed to be cleaned up. But looking at it from another angle, I wanted to make sure that the external behavior of the system was what I actually wanted before I invested more time into optimizing its implementation. I didn't have insight at this point in time to answer that question, so I decided to leave the code messy for the time being until I had a chance to see how well the new sharing mechanism performed in production. 

> HISTORY: Deployed on 2013-08-26 and then merged on 2013-08-29.
>
> [View complete diff](https://github.com/elm-city-craftworks/practicing-ruby-web/pull/173)

## Step 9: Get practicingruby.com running on our new VPS

While I kept working on the sharing mechanism, Jordan was busy setting up a new production environment for us. We set up a temporary subdomain new.practicingruby.com for this purpose, and that allowed us to do some live testing while he got everything up and running.

Technically speaking, we only need to upgrade our Ruby version in order to fix the problems we encountered with DelayedJob, so spinning up a new VPS instance purely for that purpose might sound a bit overkill at first glance. However, starting with a blank slate environment allowed us to upgrade the rest of our serverside dependencies at a relaxed pace, without worrying about potentially causing large amounts of site downtime. Spinning up the new environment in parallel before decommissioning the old one also meant that we could always switch back to our old environment if we encountered any problems during the migration to the new server.

On 2013-08-30, we decided to migrate to the new environment. The first step was to pull in the delayed broadcast mailer code and do live tests similar to the ones we had done earlier. After we found that those went smoothly, we decided to do a complete end-to-end test by using the new system to deliver an announcement about the planned maintenance downtime. That worked without any issues, and so at that time we were ready to perform the cut over.

We made sure to copy over all the data from our old environment immediately after putting it into maintenance mode, and then we updated our DNS entries to point practicingruby.com at the new server. After the DNS records propagated, we used a combination of watching our logs and our analytics dashboard (Mixpanel) to see how things were going. There were only two minor hiccups before everything got back to normal:

* We had a small issue with our Oauth configuration on Github, but resolving it was trivial after we realized that it was not in fact a DNS-related problem, but an issue on our end.

* We realized as soon as we spun up our cron jobs on the new server that our integration with Mailchimp's API had broken. The gem we were using was not Ruby 2.0 compatible, but we never realized this in development because we use it only in a tiny cleanup script that runs behind the scenes on the server. Thankfully because we had isolated this dependency from our application code, [changing it was very easy](https://github.com/elm-city-craftworks/practicing-ruby-web/pull/177/files).

These two issues were the only problems we needed to debug under pressure throughout all the work we described in this article. Given how much we changed under the hood, I am quite proud of that fact.

## Reflections

The state of practicingruby.com immediately after our server migration was roughly comparable to that of the highway construction photograph that you saw at the beginning of this article: some improvements had been made, but there was still plenty of old cruft left around, and lots of work left to be done before things could be considered finished. My goal in writing this article was not to show a beautiful end result, but instead to illustrate a process that is seldom discussed.

In the name of preserving realism, I dragged you through some of our oldest and worst code, and also showed you some newer code that isn't much nicer looking than our old stuff. Along the way, we used countless techniques that feel more like plumbing work than interesting programming work. Each step along the way, we used a different technique to glue one bit of code to another bit of code without breaking old behaviors, because there was no good one-size fits all solution to turn to. We got the job done, but we definitely got our hands dirty in the process.

I feel fairly confident that some of the changes I showed in this article are ones that I will be thankful for in the long haul, while others I will come to regret. The trouble of course is knowing which will be which, and only time and experience can get me there. But hopefully by sharing my own experiences with you, you can learn something from my mistakes, too!

> Special thanks goes to Jordan Byron (the maintainer of practicingruby.com) for collaborating with me on this article, and for helping Practicing Ruby run smoothly over the years.
