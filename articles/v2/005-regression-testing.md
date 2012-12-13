The most simple and unambiguous benefit that automated testing provides is that it can be used to minimize the impact of regressions. If each bug fix is accompanied by a corresponding test, it makes it very likely that known defects will be noticed immediately whenever they resurface. Over the lifetime of a project, regression tests end up forming a safety net that makes refactoring easier and maintenance work less scary.

In this article, we will look at a couple of real bugs that were fixed in practicingruby.com and the tests that are meant to prevent them from recurring. Working on these fixes taught Jordan and me two valuable lessons that I think are worth sharing. For some, they will serve as evidence of why you should be writing regression tests; for others, they will serve more as a reminder of why the time you're investing into your tests is worthwhile.

_As a bit of a warning to those expecting sage advice in this article, I am in no way an expert at Rails testing, and even my Ruby testing skills are probably average. But this newsletter is about practicing Ruby, not just pontificating about it. I'm not afraid to show some of my own weak spots from time to time, as long as we can all learn something from it._

### LESSON 1: More granular tests make for better regression suites  

We now have support for Github-style mentions in our comments here on practicingruby.com, but the initial version of this feature was much more fragile than the one that is now in place. When Jordan first demonstrated the feature to me, I immediately tried to break it by manually testing how it dealt with case sensitivity, punctuation, and mentions of users who were not subscribed to Practicing Ruby. I succeeded at demonstrating bad behavior for two out of three of these cases, so Jordan quickly coded up a fix and an accompanying test for that fix. The test he wrote is shown here:

```ruby
test "returns an array of valid users mentioned" do
  person = Factory(:user, :github_nickname => "PerSon")
  frank = Factory(:user, :github_nickname => "frank-pepelio")

  comment = Factory(:comment,
    :body => "I mention @person: and @FRank-pepelio but @noexist isn't there")

  mentioned_users = comment.mentioned_users

  assert_equal 2, mentioned_users.count
  assert mentioned_users.include?(person)
  assert mentioned_users.include?(frank)
end
```

Seeing this test go green was sufficient evidence for me that after the fix, the feature was working as I'd expect it to. We committed it, and for the moment, the problem was solved. But when I took a second look at this test while preparing this article, I decided that I should rewrite it. Can you guess why?

If you guessed that it was because I have some deep-rooted hatred for Frank Pepelio, unfortunately that's not correct. But if instead you guessed that it was because I thought this test was covering too many different behaviors at once, you were spot on. I came up with the following tests as a suitable replacement.

```ruby
test "returns an array of valid users mentioned" do
  person = Factory(:user, :github_nickname => "PerSon")
  frank = Factory(:user, :github_nickname => "frank-pepelio")

  comment = Factory(:comment,
    :body => "I mention @PerSon and @frank-pepelio")

  mentioned_users = comment.mentioned_users

  assert_equal 2, mentioned_users.count
  assert mentioned_users.include?(person)
  assert mentioned_users.include?(frank)
end

test "omits mentioned users that do not have a matching user record" do
  frank = Factory(:user, :github_nickname => "frank-pepelio")

  comment = Factory(:comment,
    :body => "I mention @frank-pepelio and @noexist")

  mentioned_users = comment.mentioned_users

  assert_equal [frank], mentioned_users
end

test "match mentioned users without case sensitivity" do
  frank = Factory(:user, :github_nickname => "frank-pepelio")

  comment = Factory(:comment,
    :body => "I mention @FRANK-pepelio")

  mentioned_users = comment.mentioned_users

  assert_equal [frank], mentioned_users
end

test "allows user mentions to include punctuation" do
  frank = Factory(:user, :github_nickname => "frank-pepelio")
  person = Factory(:user, :github_nickname => "person")

  comment = Factory(:comment, :body => "@person, @frank-pepelio: YAY!")
  mentioned_users = comment.mentioned_users

  assert_equal 2, mentioned_users.count
  assert mentioned_users.include?(person)
  assert mentioned_users.include?(frank)
end
```

These tests are a whole lot more verbose than the ones Jordan wrote, even though they cover the same set of behaviors. However, the extra effort of writing tests this way gains us quite a bit. Our new tests are much more granular, making the expectations for this feature more explicit and easier to understand for those reading the tests. More important, though, the increased granularity extends to feedback provided by test failures. If we make a change that causes our mentions to no longer be case sensitive, we get a failure in the specific test about case sensitivity, rather than in a catch-all test designed to fully define the behavior of the feature. 

The value of your test suite as a safety net against regressions is directly related to the granularity of your tests. For any given feature, the more complex it is, the greater the importance of granular testing. Additionally, the more features you have, the more difficult it will be to keep track of the edge cases of any single feature unless you break out each of its behaviors explicitly. Because of the multiplicative effect between these two things, it means that most of the time, it's worth it to make your test more granular, even if doing so takes a bit more time, effort, and code.

### LESSON 2: Testing your critical paths is important

Part of the process of rolling out our new notification features (including mentions) was to set up background job processing for sending emails. We decided to try out [MailHopper](https://github.com/cerebris/mailhopper), but in order to do so, we needed to upgrade to Rails 3.1. The upgrade was relatively straightfoward and Jordan did a fair bit of manual testing to make sure that notification emails were still being sent and that the application as a whole was still working as expected.

Unfortunately, the one area he forgot to check was the code I wrote for linking MailChimp subscriptions to GitHub accounts. This may be some of the most simple code in the the application, but I still managed to use a couple of deprecated APIs without realizing it. If we had some integration tests in place, we probably would have caught these issues, but because we didn't, they didn't surface until several hours after we deployed the Rails 3.1 upgrade.

The issues themselves were downright trivial. In one of our HAML-based views, I had written `- form_for` instead of `= form_for`, which upon upgrading to Rails 3.1 prevented the form from being rendered. In the process of fixing this issue, I stumbled across another problem. It used to be possible to call `ActionMailer` methods by prefixing them with `deliver_`; in Rails 3.1, though, you use an ordinary method call that returns a `Mail` object that you're expected to explicitly call `deliver` on. Because my code was using the old syntax, it ended up raising a `NoMethodError` rather than sending an email once we did the upgrade. It took only a few minutes to fix these issues, but the fact that they cropped up at all was a sign of a deeper problem.

In the early stages of building out practicingruby.com, we avoided writing tests because the code was very simple and because doing a manual check of all the features took no more than a few minutes before each deploy. I wanted to focus on content generation, and I didn't want to overwhelm Jordan with too much work on Practicing Ruby because it would take his focus off of Mendicant University. But, as we fast-forward a couple weeks, the application is now complex enough that not writing tests is costing us time rather than saving us time. Even if we still don't necessarily need 100% test coverage, we need to make sure that we've got the critical paths through this application covered so that the site can remain a good experience for our readers.

With that in mind, I set out to write integration tests that cover the process of linking your GitHub account to your MailChimp subscription from end to end. I started with the easy case of when your GitHub email address matches the one you subscribed to this newsletter with. The code below is my first rough draft of a test that walks through that process.

```ruby
test "github autolinking" do
  Factory.create(:user, :email => "gregory.t.brown@gmail.com")

  OmniAuth.config.add_mock(:github, {
    :uid => '12345',
    :user_info => {
      :nickname => 'sandal',
      :email    => "gregory.t.brown@gmail.com"
    }
  })

  visit community_url

  auth_link = Authorization.find_by_github_uid("12345").
                            authorization_link


  assert_equal authorization_link_path(auth_link), current_path

  refute_empty ActionMailer::Base.deliveries

  mail = ActionMailer::Base.deliveries.first
  ActionMailer::Base.deliveries.clear

  assert_equal ["gregory.t.brown@gmail.com"], mail.to

  visit "/sessions/link/#{auth_link.secret}"
  assert_equal articles_path, current_path
end
```

I was initially expecting this test to pass, but upon running, it ended up with a failure. I had forgotten that my previous manual testing was only of the more complex workflow, which has you explictly enter your email address. The fact that the same confirmation email gets sent by our application in two different ways is a sign that there is some violation of DRY going on, but the more immediate realization I had was that the setup costs of testing these different scenarios manually was causing both Jordan and me to cut corners and to not test all the paths we should have been testing. Fortunately, rewriting another `ActionMailer` call got this test to go green. 

No longer feeling confident in my ability to weed out these errors from memory, I did a quick projectwide search with ack on the word "deliver" to confirm that this was indeed the last instance of this particular bug. After I was sure that was the case, I was able to move onto the workflow that caused me to detect this bug in the first place: our manual MailChimp-to-GitHub linking process. It is more or less the same set of steps, but involves filling out a form field before the confirmation email gets sent. 

```ruby
test "github manual linking" do
  Factory.create(:user, :email => "gregory.t.brown@gmail.com")

  OmniAuth.config.add_mock(:github, {
    :uid => '12345',
    :user_info => {
      :nickname => 'sandal',
      :email    => "test@test.com"
    }
  })

  visit community_url

  auth_link = Authorization.find_by_github_uid("12345").
                            authorization_link


  assert_equal edit_authorization_link_path(auth_link), current_path
  fill_in "authorization_link_mailchimp_email", 
          :with => "gregory.t.brown@gmail.com"
  click_button("Link this email address to my Github account")

  refute_empty ActionMailer::Base.deliveries
  mail = ActionMailer::Base.deliveries.first
  ActionMailer::Base.deliveries.clear

  assert_equal ["gregory.t.brown@gmail.com"], mail.to

  auth_link.reload

  visit "/sessions/link/#{auth_link.secret}"
  assert_equal articles_path, current_path
end
```

Because I had already fixed the issues that would have prevented these tests from passing, this test went green without any additional modifications to the application. However, when writing regression tests, it's important to be able to verify that your tests are able to detect the defects they're meant to protect against. To do so, I went ahead and reverted each of my fixes one by one, then reapplied them to confirm that without each fix, the tests fail, and with them, the tests pass. This check isn't quite a substitute for writing tests before writing code but does at least help ensure that your tests are valid.

Now that the immediate concern of having some tests to accompany my bug fixes was resolved, I turned my eye to the messy code I had just written and found it in dire need of refactoring. Because I typically only ever worked far down the Rails stack in my consulting work, proper integration testing is a skill I never picked up. But just my sense of Ruby coding in general made me realize that I could do some simple extractions to at least hide some of the nasty stuff this code was doing. The following tests show what I ended up with.

```ruby
test "github autolinking" do
  email = "gregory.t.brown@gmail.com"
  uid   = "12345"

  create_user(:email => email)
  login(:nickname => "sandal", :email => email, :uid => uid)

  visit community_url
  get_authorization_link(uid)

  assert_confirmation_sent(email)

  assert_activated
end

test "github manual linking" do
  mailchimp_email = "gregory.t.brown@gmail.com"
  github_email    = "test@test.com"
  uid             = "12345"

  create_user(:email => mailchimp_email)
  login(:nickname => "sandal", :email => github_email, :uid => uid)

  visit community_url
  get_authorization_link(uid)

  assert_email_manually_entered(mailchimp_email)

  assert_confirmation_sent(mailchimp_email)

  assert_activated
end
```

To get the code to look like this, I didn't do anything fancy. I just did ordinary method extractions by pushing paragraphs of code down into functions. In the event that you want to see exactly what changes I made, you can check out [this gist](https://gist.github.com/3829d5bfc124c3640c5b), which contains the entire test file. The end result provides a fairly nice high-level description of each step of these two scenarios.

Writing these tests took me longer than I would have liked, in part because I'm not that comfortable with integration testing in Rails, but also because there are a lot of moving parts to consider. However, because this is a multistep process that involves nontrivial setup, repeated manual testing would quickly end up taking up more of my time than writing these tests did. For that reason, I definitely suggest writing up some integration tests for whatever critical paths you have in your applications.

To make a long story short, the more annoying your workflows are to test manually, the more important it is for you to write automated tests that do the job for you. Also, your critical paths through your application really ought to be covered with automated tests so that they don't just go up in smoke without you noticing. The good news is that your integration tests needn't be bulletproof in order to be useful.

### Reflections

I am trying to reach two audiences at once with this article: those who'd benefit from seeing how bug fixes can be accompanied by tests to help prevent regressions, and those who have experience with Rails testing who can discuss strategies for how to improve the tests I wrote.

To be brutally honest, this newsletter is called Practicing Ruby and not Practicing Rails for a reason: I'm not really a Rails developer. However, I imagine that most of our readers have at least some experience with Rails and that many of you may be much stronger at testing Rails applications than I am. My hope is that you will take this opportunity to learn by teaching and tell me a thing or two about how I could have made my integration tests better.

In particular, I am made uneasy by the large amount of global state that I'm having to manage in my integration tests. Is this a standard practice? If not, what are the alternatives? Am I supposed to use more mock objects to avoid these situations? Should I be designing my applications differently to make them easier to test? If so, what changes could I make to my implementation code to make these tests easier to write? These are all questions that ran through my mind as I was working on this article, and I'd appreciate any links to good resources that might help me answer them, or some advice based on your own experiences.
