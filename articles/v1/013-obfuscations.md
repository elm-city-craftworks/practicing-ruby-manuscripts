Rather than always focusing on SERIOUS BUSINESS, I'd like share something a little more light hearted today. Whether you celebrate Christmas or not, I think you'll find this little holiday themed hack a great deal of fun to play with.

### Christian Neukirchen's Christmas Hack

When I first started programming in Ruby, the ruby-talk mailing list was the best place to interact with the community and keep up with other active Ruby hackers. But because there were a lot more hobbyists in 2004 than there were people doing Ruby as a full time job, the posts focused on sharing fun hacks just as often as they did on discussing practical issues.

One of my favorites was [Christian Neukirchen](http://twitter.com/#!/chneukirchen)'s obfuscated Christmas message to the Ruby community in 2004. I've copied the source code below, and I encourage you to run it and see that it is indeed a valid Ruby program!

```ruby
s="IyBUaGFua3MgZm9yIGxvb2tpbmcgYXQgbXkgY29kZ
S4KIwojIENvcHlyaWdodCAoQykgMjAwMiAgQ2hyaXN0a
WFuI      E       5       l       d     Wtpc                                
mNoZ  W       4      gP       G       N obmV
1a2l      y       Y 2hlb  k       B     nbWF
pbC5  j       b    20+CiM     K       I yBUa
GlzI      H       Byb2dyYW        0     gaXM
gZnJ  l       Z  SBzb2Z0d2F   y       Z Tsge
W91I      G     NhbiByZWRpc3      R     yaWJ
1dGU  g        aXQgYW5kL29yCi M       g bW9k
aWZ5      I   Gl0IHVuZGVyIHRoZ    S     B0ZX
Jtcy  B      vZiB0aGUgR05VIEdlb       m VyYW
wgUH      V      ibGljIExpY       2     Vuc2
UuCg  p       T VERPVVQuc3lu  Y       y A9IH
RydW      U    KZDEsIGQyID0gM     C     4xNS
wgMC  4       wNgpzID0gIk1lcnJ        5 IGNo
cmlz      d  G1hcywgLi4uIGFuZCB   h     IGhh
cHB5  I     G5ldyB5ZWFyIgptID0gJ      X d7LC
AuID       ogISArICogMCBPIEB9CnUg P     SAiI
CIgK  i   BzLnNpemUKCnByaW50ICJcci    A gI3t
1fVx      y   IjsKCigwLi4ocy5z    a     XplL
TEpK  S      50b19hLnNvcnRfYnkg       e yByY
W5kI      H 0uZWFjaCB7IHxyfAogIH  N     sZWV
wIGQ  x    CiAgbmV4dCBpZiBzW3JdID     0 9ICI
gIls      wXQogIG0uZWFjaCB7IHxrfAo      gICA
gdVt  y  XSA9IGsKICAgIHByaW50ICIgIC   N 7dX1
cciI    KICAgIHNsZWVwIGQyCiAgfQogIHV    bcl0
gPSB   zW3JdCiAgcHJpbnQgIiAgI3t1fVxyI g p9Cg
pzbG  VlcCBkMgpwcmludCAiICAje3V9IVxyI   jsKc
2xlZ  X       A    gMwpwc     m       l udCA
iICA      j        e3V9IS A       g     LS1j
aHJp  c       z    JcbiI7     C       g ojIG
ZpbG      x        lciBzc G       F     jZSA
jIyM  j       I    yMjIyM     j       I yMjI
yMjI      y       M       j       I     yMjI
yMK";eval s.delete!(" \n").unpack("m*")[0]##
### Copyright (C) 2004  Christian Neukirchen
```

When run, this code prints out <i>"Merry christmas, ... and a happy new year! --chris2"</i> by randomly filling in each character in a little animation. After some folks commented on how cool this hack was, someone inevitably asked how it was done, which lead another Ruby hacker Michael Neumann to post his guess to the list. Here is what he said:

>Pretty easy (except drawing the tree :). Write the source-code first, then `base64` encode it, and insert newlines/whitespace to make the picture.

At the time, I was too much of a beginner with Ruby to fully appreciate the solution discussion, and mostly just chalked it up to magic. But now, the above statement is immediately obvious to me, and since it wasn't further explained in the mailing list thread, I can give an example for those who are in the same shoes now that I was in a few years ago.

What I didn't know at the time is that `Base64` is an encoding that allows you to translate any binary data into purely printable characters by converting the contents into a string of characters that uses basic alphanumeric values. I would have known that if I read the documentation for Ruby's `Base64` standard library, but again, I was a newbie at the time. :)

It turns out that the idea for `Base64` encoding was extracted from how MIME attachments in email are implemented. This is all stuff you can find on wikipedia, so rather than digging into the gory details, let's see how it relates to the problem at hand.

The following small snippet should clear things up a bit.

```ruby
>> source = "puts 'hello world'"
=> "puts 'hello world'"
>> encoded_source = Base64.encode64(source)
=> "cHV0cyAnaGVsbG8gd29ybGQn\n"
>> Base64.decode64(encoded_source)
=> "puts 'hello world'"
>> eval Base64.decode64(encoded_source)
hello world
=> nil
```

Another way of decoding `Base64` encoded strings is via the `String#unpack` method, using the template `"m*"`. You can see this in Christian's code, which is what tipped Michael off in the first place. With that in mind, we can build a tiny obfuscated "Hello World" program.

```ruby
s = 
"c  H   V0cyA 
 n  a     G
 VsbG     8
 g  d     2
 9  y    bGQn"

eval s.delete(" \n").unpack("m*")[0]
```

In the end, Michael was right when he said this was pretty easy to do. As long
as you understand some basic string manipulation and how to decode a `base64` 
encoded string, you could use this technique to render your code as pretty much any arbitrary ASCII art.

Of course, one would expect that the guy who eventually would go on to create something as clever and useful as the [Rack web server interface](https://github.com/rack/rack) would have an extra trick or two up his sleeve. Not to disappoint, Christian confirmed Michael's explanation was valid, but in the process revealed that he felt it'd be too fragile and tedious to manually format the code himself into the desired ascii art.

For those curious about how he got around this problem, you can check out his [full solution](http://groups.google.com/group/comp.lang.ruby/msg/aa5b4f8eaa85e6b8?dmode=source)
 which implements a code generator that fills in a template with the `base64` encoded source.

While the code should be pretty easy to follow with a little effort, feel free to post questions here if you need help figuring things out. It's a really neat bit of code and is worth exploring, so I don't mind giving some hints where needed.

### Reflections

Writing this article reminded me of two lessons that I sometimes forget, even to this day.

The first lesson is that you can't judge the complexity of something by simply scratching its surface. When I saw this code posted to ruby-talk back in 2004, even though I was a newbie at the time, I could have figured it out if I only took a bit of time to study the topics that were being discussed. But since I saw a bunch of obscure binary data in the shape of a Christmas tree being passed to `eval()`, I judged the snippet as being too complicated for me, appreciated it for its magic, and moved on. That sort of lack of self-confidence can really prevent you from stumbling upon interesting new ideas, tools, and techniques.

The second lesson is that hacking doesn't always have to be SERIOUS BUSINESS.
Because I'm working on things I feel are super important most of the time, it's
easy for me to forget to be playful and generally curious. Sometimes I feel like
I'm too busy to do something just for the joy of the hack, and that worries me a bit. 
Writing this article reminded that I should resist this temptation, and make more 
time and space in my life for playful discovery, because it is a great way to learn 
and have fun at the same time.

  
> **NOTE:** This article has also been published on the Ruby Best Practices blog. There [may be additional commentary](http://blog.rubybestpractices.com/posts/gregory/045-issue-14-obfuscations.html#disqus_thread) 
over there worth taking a look at.
