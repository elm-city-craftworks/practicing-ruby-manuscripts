One project that I've always wanted to work on is the creation of a generic table datastructure for Ruby. I've partially solved this problem in a dozen contexts before, but I've never come up with something I've been happy enough with to extract into its own gem. 

Every time I've attempted to work on this project in the past, I've set myself up for failure by thinking of the seemingly endless amount of things that a generic table structure would need to implement. Recently, I approached the problem from a different angle and ended up feeling a lot happier with the way things went. In this article, I share the lessons I learned that helped me attack this very sticky problem.

### Lesson 1: Work on specific cases before attempting to generalize

In the past, I had always been bogged down by thinking of all the possible ways my table structure was going to be used. This time around, I forced myself to think of a single, specific use case to focus on first. Instantly, the idea of of performing some manipulations on sales data came to mind, because I hate the reporting features PayPal gives me.

Typically, I'd start by creating some fake data that was themed to fit this scenario, but lately I've been experimenting more and more with trying to work with real data whenever it isn't too inconvenient. I've had mixed results with that approach, but this time around, a few minutes of cleanup work got me from a nastily formatted CSV file with way too much information to a nice array of arrays in JSON format that's concise enough to let us see the entire contents of the file, as shown here: 

    [["Date","Payments Received","Amount Received",
      "Payment Fees","Net Amount"],
     ["6/30/2011","7","52.00","-3.93","48.07"],
     ["6/29/2011","14","152.00","-8.98","143.02"],
     ["6/28/2011","5","40.00","-2.73","37.27"],
     ["6/27/2011","0","0.00","0.00","0.00"],
     ["6/26/2011","2","12.00","-0.99","11.01"],
     ["6/25/2011","1","4.00","-0.46","3.54"],
     ["6/24/2011","0","0.00","0.00","0.00"],
     ["6/23/2011","1","16.00","-0.76","15.24"],
     ["6/22/2011","2","12.00","-0.95","11.05"],
     ["6/21/2011","0","0.00","0.00","0.00"],
     ["6/20/2011","0","0.00","0.00","0.00"],
     ["6/19/2011","1","16.00","-0.76","15.24"],
     ["6/18/2011","0","0.00","0.00","0.00"],
     ["6/17/2011","1","4.00","-0.42","3.58"],
     ["6/16/2011","0","0.00","0.00","0.00"],
     ["6/15/2011","0","0.00","0.00","0.00"],
     ["6/14/2011","2","36.00","-1.69","34.31"],
     ["6/13/2011","0","0.00","0.00","0.00"],
     ["6/12/2011","0","0.00","0.00","0.00"],
     ["6/11/2011","1","4.00","-0.46","3.54"],
     ["6/10/2011","0","0.00","0.00","0.00"],
     ["6/9/2011","1","4.00","-0.46","3.54"],
     ["6/8/2011","0","0.00","0.00","0.00"],
     ["6/7/2011","1","4.00","-0.46","3.54"],
     ["6/6/2011","0","0.00","0.00","0.00"],
     ["6/5/2011","2","20.00","-1.22","18.78"],
     ["6/4/2011","4","52.00","-3.23","48.77"],
     ["6/3/2011","9","100.00","-6.13","93.87"],
     ["6/2/2011","8","72.00","-4.79","67.21"],
     ["6/1/2011","8","136.00","-6.67","129.33"]]

My next step was to come up with a question about this data that would be easy to answer by visual inspection, and trivial to represent using nothing more than primitive Ruby objects. The question I settled on was, "How many payments were received on 6/14/2011?"

    data = JSON.parse(File.read("sales.json"))
    row = data.find { |x| x[0] == "6/14/2011" }
    p row[1] #=> "2"

If instead I'd chosen a question that required too much thought to answer, the judgmental side of my brain would have kicked in too early and derailed my efforts to get even the smallest start on the problem. However, by picking an extremely simple problem to work on, I managed to turn the judge's voice in my head into a collaborator rather than an interrogator.

The judge looked at these three lines of code and instantly started in with his criticisms.

JUDGE: _"This is just terrible! If the order of the columns in your data changes, this code is going to break! Also, your output is clearly supposed to be numeric, but you're getting back a string. And that JSON call looks ugly too, and your mother is a . . . "_

Addressing all his points right away would have been a bad idea because it would have led me into a spiral of self-doubt. Instead, I just looked for one thing we could agree on so that we had some common ground to start from. The criticism about the column order dependency was a pretty good one, so I decided to work with that.

Whenever I think of good APIs that I've seen for table interactions, the approach ActiveRecord 3 takes always comes to mind. It seemed to fit this particular problem well, so I cautiously asked the judge for his opinion on the following code.

    table = Table.new(data)
    row   = table.where("Date" => "6/14/2011").first

    p row["Payments Received"] #=> "2"

JUDGE: _"Hmm . . . that `first()` call looks weird, but this is a LOT better than your last attempt. I won't be convinced until I see an implementation, though. Also, do you really think baking in the assumption that there are headers in the first row of your data is a good idea?"_

His point about headers was a good one, so—knowing that this was the closest thing I could get to approval from the judge—I started writing some tests that took his suggestion into account.

    describe "Table" do
      it "must be able to search for matching records" do
        fixture_dir = "#{File.dirname(__FILE__)}/fixtures"
        json_data   = File.read("#{fixture_dir}/sales.json")
        
        names, *data  = JSON.parse(json_data)
        table         = Table.new(data, names)

        expected_payments = "2"
        
        match = table.where("Date" => "6/14/2011").first
        match["Payments Received"].must_equal(expected_payments)
      end
    end

Before I even get a chance to run these tests, the judge snapped at me.

JUDGE: _"Whoa, that fixture loading code looks disgusting. Do you really think that you can get away with that while I'm watching?"_

He was right, of course, so I opened up my test helper file and wrote a little method to hide the messy code and isolate it to one place:

    def json_fixture(filename)
      test_dir = File.dirname(__FILE__)
      JSON.parse(File.read("#{test_dir}/fixtures/#{filename}.json"))
    end

Using this helper file, I was able to make my tests look a whole lot better.
   
    describe "Table" do
      it "must be able to search for matching records" do
        names, *data  = json_fixture("sales")
        table         = Table.new(data, names)

        expected_payments = "2"
        
        match = table.where("Date" => "6/14/2011").first
        match["Payments Received"].must_equal(expected_payments)
      end
    end

I looked to the judge for approval and got a half-hearted shrug, which was good enough for me. By the time I finished writing these tests, I already had an idea in mind for how to implement a solution.

    class Table
      def initialize(data, attribute_names)
        @data = data.map { |e| Hash[attribute_names.zip(e)] }
      end

      def where(conditions)
        @data.select do |row|
          conditions.all? { |k,v| row[k] == v }
        end
      end
    end

Even though this code passed the test, the judge could no longer contain himself, and fired off another burst of scathing criticism.

JUDGE: _"This code is a giant hack. You store each row in a hash, but hashes are meant for unordered content and a row is necessarily ordered. Yes, I know that in Ruby 1.9 you can iterate over hashes in insert order, but that's going to come back and bite you later. What if you want to introduce a column rename operation in the future? There is no way to do that with your current code without either changing the iteration order or generating entirely new hashes for the entire structure. Even worse, hash keys must be unique. What if you have a table with two column names that are the same? To make matters worse, the code reeks of primitive obsession. Unless you introduce a `Row` object soon, every feature you add to `Table` is going to get more and more complicated because it will have two concerns: representing an ordered list of rows and manipulating the data within those rows. But who says that users are going to want to work with just rows? What if they want to organize their data by columns instead? Oh, and as I was saying about your mom . . . "_

This rant was too much to take in all at once, and I felt overwhelmed. I knew that responding directly to his criticisms line by line would only fan the flames. The points he made about problems I might run into later could have very well been valid, but thinking about them at this particular point in time would have sent me down a deadly path of feature creep. I needed to take a step back and consider the big picture.

### Lesson 2: Seek ways to defer tough design decisions

After catching my breath, I realized that I could address many of the judge's points without actually making any major decisions about implementation details. I could do this by introducing a `Record` object. This object would initially have a core implementation similar to my previous example but would make it much easier to introduce changes down the line. The following test describes what I was shooting for:

    describe "Record" do
      it "must allow access to attributes by name" do
        data       = ["6/14/2011", "2", "36.00", "-1.69", "34.31"]
        names      = ["Date", "Payments Received", "Amount Received", 
                      "Payment Fees", "Net Amount"] 

        record     = Record.new(data, names)

        record["Payments Received"].must_equal("2")
        record["Payment Fees"].must_equal("-1.69")
      end

      it "must support conditional matching" do
        data       = ["6/14/2011", "2", "36.00", "-1.69", "34.31"]
        names      = ["Date", "Payments Received", "Amount Received", 
                      "Payment Fees", "Net Amount"] 

        record     = Record.new(data, names)

        record.matches?("Date" => "6/14/2011").must_equal(true)
        record.matches?("Date" => "6/12/2011").must_equal(false)
      end
    end

To pass these tests, I pushed logic down from the `Table` class into a newly created `Record` class:

    class Record
      def initialize(values, attribute_names)
        @data = Hash[attribute_names.zip(values)]
      end

      def [](key)
        @data[key]
      end

      def matches?(conditions)
        conditions.all? { |k,v| self[k] == v }
      end
    end

The judge was eerily silent as this test went green. Perhaps he was waiting to see what my next move would be, or maybe he had just run out of jokes about my mom. Nonetheless, I took this as my cue to go update my `Table` code so that it would use `Record` objects instead of hashes:

    class Table
      def initialize(data, attribute_names)
        @data = data.map { |e| Record.new(e, attribute_names) }
      end

      def where(conditions)
        @data.select { |record| record.matches?(conditions) }
      end
    end

Just after I reran my `Table` tests and saw that they were passing, the judge said something that I had to ask him to repeat, because it was so surprising.

JUDGE: _"Not bad."_

He refused to elaborate, but I think I finally figured out why he approved of this newer version of my code. After thinking about his previous barrage of complaints, I realized that this new `Table` implementation did not raise any of the same questions. When you look at the problem at hand and then at the object that directly solves that problem, the object looks natural, well focused, and unsurprising. A `Table` is a collection of `Record` objects. The `Table#where` method selects from that collection the records that match the conditions. These explanations are very easy to follow and directly line up with the code itself.

The thing that made this new `Table` code "not bad" in the eyes of the judge is that it is a proper abstraction, whereas my previous implementation was just an indirect wrapper over some primitive operations. My newer code was written with the changing needs for our `Record` object in mind, which is what made all the difference.

### Lesson 3: Let real use cases be your guide, not imagined scenarios

In his various outbursts, the judge pointed out lots of different things that he felt my `Table` code should do. Here is a short list of them, for those who haven't been keeping track:

* Should be able to set column data types (i.e., convert a column that contains numeric strings into numeric values)
* Should be able to deserialize array-of-arrays datastructure from JSON
* Should provide a way to match a single record rather than calling `first()` on the array returned by `Table#where`
* Should preserve data ordering explicitly
* Should support both by column and by row access
* Should be able to rename columns
* Should be able to support multiple columns with the same name

On their own, all these ideas sound like good ones. But taken together, we're talking about a lot of additional work for a nonspecific gain. I know from previous experience that going down this path will lead to a very complex, very bloated object. Take a look at my `Ruport::Data::Table` implementation from several years ago if you want to see just how complicated this sort of thing can get:

https://github.com/ruport/ruport/blob/master/lib/ruport/data/table.rb 

This time around, I'm going to take a very different approach, adding features to my `Table` object only when I have a real project that depends on that feature. Even when there is something the API doesn't support, I will try to work around the problem and see how much pain it causes me. Only after something causes me a lot of pain in one place or a little pain spread across several places will I add new features.

I am very interested to see what kind of code is produced from this sort of aggressive use-case-driven development. But testing this idea is something that we're going to need to do collectively as homework, because not enough time passed between when I started this experiment and when I published the article you are reading now. 

If you'd like to help, please take a look at the following repository on Github and follow the instructions in the README.

http://github.com/sandal/waffle

### Reflections

Hopefully, by following in my footsteps, you were able to notice some similarities to your own struggles with sticky projects. As this article was just a story about an approach that seems to have worked for me, your mileage will probably vary. Still, I'd love to hear what you think of the ideas I mentioned here, especially if you have a different way of dealing with this kind of problem.
