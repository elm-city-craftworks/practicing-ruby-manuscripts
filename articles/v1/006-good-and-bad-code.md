In this issue and the next one, I'd like to demonstrate one of my favorite learning exercises while inviting you to follow along at home. It's something I usually do while teaching in a one-on-one setting, but I think we can adapt it for a broader audience and still get a lot out of it.

In this exercise, the goal is to first produce some bad code, and then steadily improve it while explaining why each change is an improvement. I usually start with a very simple problem but then add some twists about how to implement it to make sure it comes out pretty bad.

One surefire way of writing bad code without resorting to intentionally writing things worse than they should be is to eliminate a few of Ruby's key organizational tools. In particular, if you want to write ugly code without it seeming fake, it is easy to do so if you never write any user defined functions, classes, or modules. So we'll do exactly that!

### Implementing Tic-Tac-Toe as a single procedure.

I've chosen the game [Tic-Tac-Toe](http://en.wikipedia.org/wiki/Tic-Tac-Toe) as the problem to focus on, because it only involves a few simple rules and can be implemented by anyone who has basic programming skills.

In fact, if you ignore end game conditions and error handling, you can get a simple prompt for a two player game with just a few lines of Ruby.

```ruby
board = [[nil,nil,nil],
         [nil,nil,nil],
         [nil,nil,nil]]

players = [:X, :O].cycle

loop do
  current_player = players.next
  puts board.map { |row| row.map { |e| e || " " }.join("|") }.join("\n")
  print "\n>> "
  row, col = gets.split.map { |e| e.to_i }
  puts
  board[row][col] = current_player
end
```

But of course, the devil is in the details. To get a fully playable game, you need some basic error checking to ensure that you can't play out of bounds or on top of another player's marker. You also need to figure out when a player has won, and when the game has ended in a draw. While this doesn't sound like a lot of work, you'll see in the code below how much complexity these simple changes add.

```ruby
board   = [[nil,nil,nil],
           [nil,nil,nil],
           [nil,nil,nil]]

left_diagonal  = [[0,0],[1,1],[2,2]]
right_diagonal = [[2,0],[1,1],[0,2]]

players = [:X, :O].cycle

current_player = players.next

loop do
  puts board.map { |row| row.map { |e| e || " " }.join("|") }.join("\n")
  print "\n>> "
  row, col = gets.split.map { |e| e.to_i }
  puts

  begin
    cell_contents = board.fetch(row).fetch(col)
  rescue IndexError
    puts "Out of bounds, try another position"
    next
  end

  if cell_contents
    puts "Cell occupied, try another position"
    next
  end

  board[row][col] = current_player

  lines = []

  [left_diagonal, right_diagonal].each do |line|
    lines << line if line.include?([row,col])
  end

  lines << (0..2).map { |c1| [row, c1] }
  lines << (0..2).map { |r1| [r1, col] }

  win = lines.any? do |line|
    line.all? { |row,col| board[row][col] == current_player }
  end

  if win
    puts "#{current_player} wins!"
    exit
  end

  if board.flatten.compact.length == 9
    puts "It's a draw!"
    exit
  end

  current_player = players.next
end
```

While relatively short, you need to read through the whole script to really understand how any part of it operates. Of course, this script did not spring together fully formed, there was a thought process that drove it to this final implementation. For those curious, you can [follow my stream of consciousness notes](https://gist.github.com/24ef3c8209877c1946bb) about what I was building and why in a step by step fashion.

Seeing these notes will hopefully give you a bit of a sense of how this process might have gone if we were pair programming on this project, working in tiny iterations to push forward just a little bit farther each time. If so, you might already be catching a glimpse of what this exercise is all about. Otherwise, there is still more for us to do!

### What Happens Next?

I've placed my bad tictactoe example in a [repository on github](https://github.com/sandal/tictactoe/tree/7fd72a33aec33f75909d8c9d59a43423b0f66b24). If you'd like to participate, please fork this repository and make one change to the code at a time, leaving detailed reasoning in each commit message as to why you're making the change. Once you're happy with what you've got, post a link in the comments section on this post so others can check out what you have done.

In the next issue, I will post my own iterative set of improvements, as well as links to some reader submissions. I will also summarize the lessons that can be learned from using this technique, and provide a few suggestions for other problems to attempt in this fashion.

### Reflections

Please leave any questions, thoughts, or suggestions in the comments section below. These articles are much better when they're treated as discussions rather than monologues. 
  
> **NOTE:** This article has also been published on the Ruby Best Practices blog. There [may be additional commentary](http://blog.rubybestpractices.com/posts/gregory/035-issue-6-good-and-bad-code.html#disqus_thread) 
over there worth taking a look at.
