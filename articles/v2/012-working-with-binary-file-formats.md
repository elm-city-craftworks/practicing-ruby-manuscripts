Most of us enjoy Ruby because it allows us to express our thoughts without worrying too much about low-level computing concepts. With that in mind, it may come as a surprise that Ruby provides a number of tools specifically designed for making low-level computing easy. In this article we'll use a few of those tools to encode and decode binary files, demonstrating just how easy Ruby makes it to get down to the realm of bits and bytes.

In the examples that follow, we will be working with the bitmap image format. I chose this format because it has a simple structure and is well documented. Despite the fact that you'll probably never need to work with bitmap images at all in your day-to-day work, the concepts involved in both reading and writing a BMP file are pretty much the same as any other file format you'll encounter. For this reason, you're encouraged to focus on the techniques being demonstrated rather than the implementation details of the file format as you read through this article.

_NOTE: This article assumes an understanding of some low level computing concepts such as the way integers are represented in binary format, including the difference between signed/unsigned integers and little/big endian byte order. You may want to watch this [introductory screencast](http://www.youtube.com/watch?v=BLnOD1qC-Vo&feature=player_detailpage#t=112s) if you don't already have a firm computer science background. I myself am not an expert in these topics, so that makes it even harder for me to explain them on the fly._

### The anatomy of a bitmap

A bitmap file consists of several sections of metadata followed by a pixel array that represents the color and position of every pixel in the image. 

While the information contained within a bitmap file is easy to process, its contents can appear cryptic due to the fact that the data in the file is encoded as a non-textual sequence of bytes. The example below demonstrates that even if you break the sequence on its section and field boundaries, it would still be a real challenge to understand without any documentation handy:

```ruby
# coding: binary

hex_data = %w[
  42 4D 
  46 00 00 00 
  00 00 
  00 00 
  36 00 00 00

  28 00 00 00 
  02 00 00 00 
  02 00 00 00 
  01 00 
  18 00 
  00 00 00 00 
  10 00 00 00 
  13 0B 00 00 
  13 0B 00 00
  00 00 00 00 
  00 00 00 00

  00 00 FF
  FF FF FF 
  00 00 
  FF 00 00 
  00 FF 00 
  00 00
]

out = hex_data.each_with_object("") { |e,s| s << Integer("0x#{e}") }

File.binwrite("example1.bmp", out)
```

However, if you understand what each field is meant to represent, the values begin to make a whole lot more sense. For example, if you know that this is a 24-bit per pixel image that is two pixels wide, and two pixels high, you might be able to make sense of the pixel array data. Below I've listed the part of the file which represents that information so that you can take a closer look.

```
00 00 FF
FF FF FF 
00 00 
FF 00 00 
00 FF 00 
00 00
```

If you run the example script and open the image file, you will see something similar to what is shown below once you zoom in close enough to see the individual pixels:

<div align="center">
  <img src="http://i.imgur.com/XhKW1.png">
</div>

By experimenting a bit with changing some of the values in the pixel array by hand, you will fairly quickly discover the overall structure of the array and the way pixels are represented. After figuring this out, you might also be able to look back on the rest of the file and determine what a few of the fields in the headers are without looking at the documentation.

After exploring a bit on your own, you should check out the [field-by-field walkthrough of a 2x2 bitmap file](http://en.wikipedia.org/wiki/BMP_file_format#Example_1) that this example was based on. The information in that table is pretty much all you'll need to know in order to make sense of the bitmap reader and writer implementations I've built for this article.

### Encoding a bitmap image

Now that you've seen what a bitmap looks like in its raw form, I can demonstrate how to build a simple encoder object that allows you to generate bitmap images in a much more convenient way. In particular, I'm going to show what I did to get the following code to output the same image that we rendered via a raw sequence of bytes earlier.

```ruby
bmp = BMP::Writer.new(2,2)

# NOTE: Bitmap encodes pixels in BGR format, not RGB!
bmp[0,0] = "ff0000"
bmp[1,0] = "00ff00"
bmp[0,1] = "0000ff"
bmp[1,1] = "ffffff"

bmp.save_as("example_generated.bmp")
```

Like most binary formats, the bitmap format has a tremendous amount of options that make building a complete implementation a whole lot more complicated than just building a tool which is suitable for generating a single type of image. I realized shortly after skimming the format description that you can skip out on a lot of the boilerplate information if you stick to 24bit-per-pixel images, so I decided to do exactly that.

Looking at the implementation from the outside-in, it's easy to see the general structure I laid out for the object. Pixels are stored as a boring array of arrays, and all the interesting things happen at the time you write the image out to file.

```ruby
class BMP 
  class Writer
    def initialize(width, height)
      @width, @height = width, height

      @pixels = Array.new(@height) { Array.new(@width) { "000000" } }
    end

    def []=(x,y,value)
      @pixels[y][x] = value
    end

    def save_as(filename)
      File.open(filename, "wb") do |file|
        write_bmp_file_header(file)
        write_dib_header(file)
        write_pixel_array(file)
      end
    end

    # ... rest of implementation details omitted for now ...
  end
end
```

All bitmap files start out with the bitmap file header, which consists of the following things:

* A two character signature to indicate the file is a bitmap file (typically "BM").
* A 32bit unsigned little-endian integer representing the size of the file itself.
* A pair of 16bit unsigned little-endian integers reserved for application specific uses.
* A 32bit unsigned little-endian integer representing the offset to where the pixel array starts in the file.

The following code shows how `BMP::Writer` builds up this header and writes it to file:

```ruby
class BMP 
  class Writer
    PIXEL_ARRAY_OFFSET = 54
    BITS_PER_PIXEL     = 24

    # ... rest of code as before ...

    def write_bmp_file_header(file)
      file << ["BM", file_size, 0, 0, PIXEL_ARRAY_OFFSET].pack("A2Vv2V")
    end

    def file_size
      PIXEL_ARRAY_OFFSET + pixel_array_size 
    end

    def pixel_array_size
      ((BITS_PER_PIXEL*@width)/32.0).ceil*4*@height
    end
  end
end
```

Out of the five fields in this header, only the file size ended up being dynamic. I was able to treat the pixel array offset as a constant because the headers for 24 bit color images take up a fixed amount of space. The [computations I used for the file size](http://en.wikipedia.org/wiki/BMP_file_format#Pixel_storage) are taken directly from wikipedia, and will make sense a bit later once we examine the way that the pixel array gets encoded.

The tool that makes it possible for us to convert these various field values into binary sequences in such a convenient way is `Array#pack`. If you note that the calculated file size of a 2x2 bitmap is 70, it becomes clear what `pack` is actually doing for us when we examine the byte by byte values in the following example:

```ruby
header = ["BM", 70, 0, 0, 54].pack("A2Vv2V") 
p header.bytes.map { |e| e.to_s(16).rjust(2,"0")  }

=begin expected output (NOTE: reformatted below for easier reading)
  ["42", "4d", 
   "46", "00", "00", "00", 
   "00", "00", 
   "00", "00", 
   "36", "00", "00", "00"]
=end
```
The sequence exactly matches that of our reference image, which indicates that the proper bitmap file header is being generated by this statement. This means that `Array#pack` is converting our Ruby strings and fixnums into their properly sized binary representations, in the format that we need them in. If we decompose the template string, it becomes easier to see where things line up:

```
  "A2" -> arbitrary binary string of width 2 (packs "BM" as: 42 4d)
  "V"  -> a 32bit unsigned little endian int (packs 70 as: 46 00 00 00)
  "v2" -> two 16bit unsigned little endian ints (packs 0, 0 as: 00 00 00 00)
  "V"  -> a 32bit unsigned little endian int (packs 54 as: 36 00 00 00)
```

While I went to the effort of expanding out the byte sequences to make it easier to see what is going on, you don't typically need to do this at all while working with `Array#pack` as long as you craft your template strings carefully. Of course, some knowledge of the underlying binary data doesn't hurt, and our implementation of `write_dib_header` actually depends on it.

```ruby
class BMP 
  class Writer
    DIB_HEADER_SIZE    = 40
    PIXELS_PER_METER   = 2835 # 2835 pixels per meter is basically 72dpi

    # ... other code as before ...

   def write_dib_header(file)
      file << [DIB_HEADER_SIZE, @width, @height, 1, BITS_PER_PIXEL,
               0, pixel_array_size, PIXELS_PER_METER, PIXELS_PER_METER, 
               0, 0].pack("V3v2V6")
    end
  end
end
```

The DIB header itself is kind of boring because is nothing more than a collection of constants combined with some values that we already computed for the BMP file header. However, if you look closely at the file format description, you'll see that our pattern doesn't actually match the datatypes of some of the fields: width, height, and horizontal/vertical resolution are all specified as signed integers, but yet we're treating them as unsigned values. This was an intentional design decision, to work around a limitation of `pack` in Ruby versions earlier than 1.9.3.

The problem is that all versions of Ruby before 1.9.3, `Array#pack` does not provide a syntax for specifying the endianness of a signed integer. While encoding negative values as unsigned integers actually seems to produce the right byte sequences for a signed integer, I'm pretty sure that's an undefined behavior. What's worse: even if you do encode the binary sequences correctly, there is no way to extract them later without [resorting to weird hacks](http://stackoverflow.com/questions/5236059/unpack-signed-little-endian-in-ruby). With this in mind, I decided to take a closer look at the particular problem to see if I had any other options.

Even though the specification says that the dimensions and resolution of the image can be negative, I have no idea how that would ever be useful. It's also really unlikely that you'll have a value large enough to overflow back into negative numbers for any of these values. For this reason, it's safe to say that you can treat these values as unsigned integers without many consequences.  This is why I used the pattern `"V3v2V6"` without much fear of bad behavior. If I only cared about supporting Ruby 1.9.3, I could have used a different pattern which DOES take in account the endianness of signed integers: `"Vl<2v2V2l<2V2"`. However, this is just trading one edge case for another, and since this is just a demo application, I went with what is more likely to work on the Ruby you're running right now.

With this weirdness out of the way, I was able to move on to working on the pixel array, which was relatively straightforward to implement.

```ruby
class BMP 
  class Writer
    # .. other code as before ...

    def write_pixel_array(file)
      @pixels.reverse_each do |row|
        row.each do |color|
          file << pixel_binstring(color)
        end

        file << row_padding
      end
    end

    def pixel_binstring(rgb_string)
      raise ArgumentError unless rgb_string =~ /\A\h{6}\z/
      [rgb_string].pack("H6")
    end

    def row_padding
      "\x0" * (@width % 4)
    end
  end
end
```

The most interesting thing to note about this code is that each row of pixels ends up getting padded with some null characters. This is to ensure that each row of pixels is aligned on WORD boundaries (4 byte sequences). This is a semi-arbitrary limitation that has to do with file storage constraints, but things like this are common in binary files. 

The calculations below show how much padding is needed to bring rows of various widths up to a multiple of 4, and explains how I derived the computation for the `row_padding` method:

```
Width 2 : 2 * 3 Bytes per pixel = 6 bytes  + 2 padding  = 8
Width 3 : 3 * 3 Bytes per pixel = 9 bytes  + 3 padding  = 12
Width 4 : 4 * 3 Bytes per pixel = 12 bytes + 0 padding  = 12
Width 5 : 5 * 3 Bytes per pixel = 15 bytes + 1 padding  = 16
Width 6 : 6 * 3 Bytes per pixel = 18 bytes + 2 padding  = 20
Width 7 : 7 * 3 Bytes per pixel = 21 bytes + 3 padding  = 24
...
```

Sometimes calculations like this are provided for you in the documentation, sometimes you need to derive them yourself. However, the deeply structured nature of most binary files makes this easy enough to do, especially if you apply some constraints to your implementation as I did here. Choosing to work with 24bit per pixel images exclusively allowed me to skirt the question of how to generalize this computation to an arbitrary amount of bits per pixel.

While the padding code is definitely the most interesting aspect of the pixel array, there are a couple other details about this implementation worth discussing. In particular, we should take a closer look at the `pixel_binstring` method:

```ruby
def pixel_binstring(rgb_string)
  raise ArgumentError unless rgb_string =~ /\A\h{6}\z/
  [rgb_string].pack("H6")
end
```

This is the method that converts the values we set in the pixel array via lines like `bmp[0,0] = "ff0000"` into actual binary sequences. It starts by matching the string with a regex to ensure that the input string is a valid sequence of 6 hexadecimal digits. If the validation succeeds, it then packs those values into a binary sequence, creating a string with three bytes in it. The example below should make it clear what is going on here:

```
>> ["ffa0ff"].pack("H6").bytes.to_a
=> [255, 160, 255]
```

This makes it possible to specify color values directly in hexadecimal strings and then convert them to their numeric value just before they get written to the file.

With this last detail explained, you should now understand how to build a functional bitmap encoder for writing 24bit color images. If seeing things broken out step by step caused you to lose a sense of the big picture, you can check out the [full source code for this object](https://gist.github.com/1351737). Feel free to play around with it a bit before moving on to the next section: the best way to learn is to actually run these code samples and try to extend them and/or break them in various ways.

### Decoding a bitmap image

As you might expect, there is a nice symmetry between encoding and decoding binary files. To show just to what extent this is the case, I will walk you through the code which makes the following example run:

```ruby
bmp = BMP::Reader.new("example1.bmp")
p bmp.width  #=> 2
p bmp.height #=> 2

p bmp[0,0] #=> "ff0000"   
p bmp[1,0] #=> "00ff00" 
p bmp[0,1] #=> "0000ff" 
p bmp[1,1] #=> "ffffff" 
```

The general structure of `BMP::Reader` ended up being quite similar to what I did for `BMP::Writer`. The code below shows the methods which define the public interface:

```ruby
class BMP
  class Reader
    def initialize(bmp_filename) 
      File.open(bmp_filename, "rb") do |file|
        read_bmp_header(file) # does some validations
        read_dib_header(file) # sets @width, @height
        read_pixels(file)     # populates the @pixels array
      end
    end

    attr_reader :width, :height

    def [](x,y)
      @pixels[y][x]
    end
  end
end
```

This time, we still are working with an ordinary array of arrays to store the pixel data, and most of the work gets done as soon as the file is read in the constructor. Because I decided to support only a single image type, most of the work of reading the headers is just for validation purposes. In fact, the `read_bmp_header` method does nothing more than some basic sanity checking, as shown below.

```ruby
class BMP
  class Reader
    PIXEL_ARRAY_OFFSET = 54

    # ...other code as before ...

    def read_bmp_header(file)
      header = file.read(14)
      magic_number, file_size, reserved1,
      reserved2, array_location = header.unpack("A2Vv2V")
      
      fail "Not a bitmap file!" unless magic_number == "BM"

      unless file.size == file_size
        fail "Corrupted bitmap: File size is not as expected" 
      end

      unless array_location == PIXEL_ARRAY_OFFSET
        fail "Unsupported bitmap: pixel array does not start where expected"
      end
    end
  end
end
```

The key thing to notice about this code is that it reads from the file just the bytes it needs in order to parse the header. This makes it possible to validate a very large file without loading much data into memory. Reading entire files into memory is rarely a good idea, and this is especially true when it comes to binary data because doing so will actually make your job harder rather than easier. 

Once the header data is loaded into a string, the `String#unpack` method is used to extract some values from it. Notice here how `String#unpack` uses the same template syntax as `Array#pack` and simply provides the inverse operation. While the `pack` operation converts an array of values into a string of binary data, the `unpack` operation converts a binary string into an array of processed values. This allows us to recover the information packed into the bitmap file header as Ruby strings and fixnums.

Once these values have been converted into Ruby objects, it's easy to do some ordinary comparisons to check to see if they're what we'd expect them to be. These sort of validations may seem trivial and a bit contrived, but they are a good way to detect corrupted files, and so are an important part of writing any decoder for binary file formats. If you do not do this sort of sanity checking, it's entirely possible that you will end up finding subtle errors later on that will be much harder to debug.

As you might expect, the implementation of `read_dib_header` involves more of the same sort of extractions and validations. It also sets the `@width` and `@height` variables, which we use later to determine how to traverse the encoded pixel array.

```ruby
class BMP 
  class Reader
    # ... other code as before ...

    BITS_PER_PIXEL     = 24
    DIB_HEADER_SIZE    = 40

    def read_dib_header(file)
      header = file.read(40)

      header_size, width, height, planes, bits_per_pixel, 
      compression_method, image_size, hres, 
      vres, n_colors, i_colors = header.unpack("V3v2V6") 

      # Note: the right pattern to use is actually "Vl<2v2V2l<2V2",
      # but that only works on Ruby 1.9.3+

      unless header_size == DIB_HEADER_SIZE
        fail "Corrupted bitmap: DIB header does not match expected size"
      end

      unless planes == 1
        fail "Corrupted bitmap: Expected 1 plane, got #{planes}"
      end

      unless bits_per_pixel == BITS_PER_PIXEL
        fail "#{bits_per_pixel} bits per pixel bitmaps are not supported"
      end

      unless compression_method == 0
        fail "Bitmap compression not supported"
      end

      unless image_size + PIXEL_ARRAY_OFFSET == file.size
        fail "Corrupted bitmap: pixel array size isn't as expected"
      end

      @width, @height = width, height
    end
  end
end
```

Beyond what has already been said about this example and the DIB header itself, there isn't much more to discuss about this particular method. That means we can finally take a look at how `BMP::Reader` converts the encoded pixel array into a nested Ruby array structure.

```ruby
class BMP 
  class Reader
    def read_pixels(file)
      @pixels = Array.new(@height) { Array.new(@width) }

      (@height-1).downto(0) do |y|
        0.upto(@width - 1) do |x|
          @pixels[y][x] = file.read(3).unpack("H6").first
        end
        advance_to_next_row(file)
      end
    end

    def advance_to_next_row(file)
      padding_bytes = @width % 4
      return if padding_bytes == 0

      file.pos += padding_bytes
    end
  end
end
```

One interesting aspect of this code is that it uses explicit numerical iterators. These are relatively rare in idiomatic Ruby, but I did not see a better way to approach this particular problem. Rows are listed in the pixel array from the bottom up, while the image itself still gets indexed from the top down (with 0 at the top). This makes it necessary to iterate over the row numbers in reverse order, and the use of `downto` is the best way I could find to do that.

The other thing worth noticing about this code is that in the `advance_to_next_row` method, we actually move the pointer ahead in the file rather than reading the padding bytes between each row. This makes little difference when you're dealing with a maximum of three bytes of padding per row (two in this case), but is a good practice for writing more efficient code that consumes less memory.

When you take all these code examples and glue them together into a single class definition, you'll end up with a `BMP::Reader` object that is capable giving you the width and height of a 24bit BMP image as well as the color of each and every pixel in the image. For those who'd like to experiment further, the [full source code for this object](https://gist.github.com/1352294) is available.

### Reflections

The thing that makes me appreciate binary file formats is that if you just learn a few basic computer science concepts, there are few things that could be more fundamentally simple to work with. However, simple does not necessarily mean "easy", and in the process of writing this article I realized that some aspects of binary file processing are not quite as trivial or intuitive as I originally thought they were.

What I can say is that this kind of work gets a whole lot easier with practice. Due to my work on [Prawn](http://github.com/sandal/prawn) I have written implementations for various different binary formats including PDF, PNG, JPG, and TTF. These formats each have their differences, but a common core of similarities makes it so that I can say with a reasonable amount of confidence that if you fully understand the examples in this article, then you are already well on your way to tackling pretty much any binary file format.

There are two things that make working with binary file formats a bit challenging for the average Rubyist. The first issue that working on this kind of programming requires knowledge of some low level concepts that you might not be familiar with if you don't come from a computer science background. However, from one self-taught person to another I can tell you that this knowledge can easily be attained in a few sittings, and that it's well worth doing so. Assuming you get past that hurdle, the second challenge is to develop a level of precision and attention to detail that greatly exceeds what is typically needed for day to day Ruby application development. 

While software development is about much more than computing, binary file processing brings you into the computer's realm and is much less forgiving than other kinds of programming. Formally trained computer science students or folks who have programmed (successfully) in C before won't have problems with developing this kind of discipline, but it was a challenge for me coming from a hack-and-slash Perl background.

But these two challenges are exactly why I encourage you to continue to study binary file formats and play around with other low level features to Ruby. The barrier-to-entry is lower than that of C, but it will still expose you to ways of thinking that will be very good for you.
