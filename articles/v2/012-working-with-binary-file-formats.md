Even if we rarely give them much thought, binary file formats are everywhere.
Ranging from images to audio files to nearly every other sort of media you can
imagine, binary files are used because they are an efficient way of
storing information in a ready-to-process format.

Despite their usefulness, binary files are cryptic and appear to be 
difficult to understand on the surface. Unlike a
text-based data format, simply looking at a binary file won't give you any 
hints about what its contents are. To even begin to understand a binary
encoded file, you need to read its format specification. These specifications 
tend to include lots of details about obscure edge cases, and that makes for
challenging reading unless you already have spent a fair amount of time 
working in the realm of bits and bytes. For these reasons, it's probably better
to learn by example rather than taking a more formal approach.

In this article, I will show you how to encode and decode the bitmap image
format. Bitmap images have a simple structure, and the format is well documented. 
Despite the fact that you'll probably never need to work with bitmap images 
at all in your day-to-day work, the concepts involved in both reading and 
writing a BMP file are pretty much the same as any other file format you'll encounter.

### The anatomy of a bitmap

A bitmap file consists of several sections of metadata followed by a pixel array that represents the color and position of every pixel in the image. 
The example below demonstrates that even if you break the sequence up into its different parts, it would still be a real 
challenge to understand without any documentation handy:

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

Once you learn what each section represents, you can start
to interpret the data. For example, if you know that this is a
24-bit per pixel image that is two pixels wide, and two pixels high, you might
be able to make sense of the pixel array data shown below:

```
00 00 FF
FF FF FF 
00 00 
FF 00 00 
00 FF 00 
00 00
```

If you run this example script and open the image file it produces, you'll see
something similar to what is shown below once you zoom in close enough to see
its pixels:

![Pixels](http://i.imgur.com/XhKW1.png)


By experimenting with changing some of the values in the pixel array by hand, you will fairly quickly discover the overall structure of the array and the way pixels are represented. After figuring this out, you might also be able to look back on the rest of the file and determine what a few of the fields in the headers are without looking at the documentation.

After exploring a bit on your own, you should check out the [field-by-field walkthrough of a 2x2 bitmap file](http://en.wikipedia.org/wiki/BMP_file_format#Example_1) that this example was based on. The information in that table is pretty much all you'll need to know in order to make sense of the bitmap reader and writer implementations I've built for this article.

### Encoding a bitmap image

Now that you've seen what a bitmap looks like in its raw form, I can demonstrate
how to build a simple encoder object that allows you to generate bitmap images
in a much more convenient way. In particular, I'm going to show what I did to
get the following code to output the same image that we rendered via a raw
sequence of bytes earlier:

```ruby
bmp = BMP::Writer.new(2,2)

# NOTE: Bitmap encodes pixels in BGR format, not RGB!
bmp[0,0] = "ff0000"
bmp[1,0] = "00ff00"
bmp[0,1] = "0000ff"
bmp[1,1] = "ffffff"

bmp.save_as("example_generated.bmp")
```

Like most binary formats, the bitmap format has a tremendous amount of options
that make building a complete implementation a whole lot more complicated than
just building a tool which is suitable for generating a single type of image. I
realized shortly after skimming the format description that you can skip out on
a lot of the boilerplate information if you stick to 24bit-per-pixel images, so
I decided to do exactly that.

Looking at the implementation from the outside-in, you can see the general
structure of the `BMP::Writer` class. Pixels are stored in a two-dimensional
array, and all the interesting things happen at the time you write the image out
to file:

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

All bitmap files start out with the bitmap file header, which consists of the
following things:

* A two character signature to indicate the file is a bitmap file (typically "BM").
* A 32bit unsigned little-endian integer representing the size of the file itself.
* A pair of 16bit unsigned little-endian integers reserved for application specific uses.
* A 32bit unsigned little-endian integer representing the offset to where the pixel array starts in the file.

The following code shows how `BMP::Writer` builds up this header and writes it
to file:

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

Out of the five fields in this header, only the file size ended up being
dynamic. I was able to treat the pixel array offset as a constant because the
headers for 24 bit color images take up a fixed amount of space. The file size
computations[^1] will make sense later once we examine the way that the pixel 
array gets encoded.

The tool that makes it possible for us to convert these various field values
into binary sequences is `Array#pack`. If you note that the file size of our
reference image is 2x2 bitmap is 70 bytes, it becomes clear what `pack`
is actually doing for us when we examine the byte by byte values 
in the following example:

```ruby
header = ["BM", 70, 0, 0, 54].pack("A2Vv2V") 
p header.bytes.map { |e| "%.2x" % e }

=begin expected output (NOTE: reformatted below for easier reading)
  ["42", "4d", 
   "46", "00", "00", "00", 
   "00", "00", 
   "00", "00", 
   "36", "00", "00", "00"]
=end
```
The byte sequence for the file header exactly matches that of our reference image, 
which indicates that the proper bitmap file header is being generated. 
Below I've listed out how each field in the header encoded:

```
  "A2" -> arbitrary binary string of width 2 (packs "BM" as: 42 4d)
  "V"  -> a 32bit unsigned little endian int (packs 70 as: 46 00 00 00)
  "v2" -> two 16bit unsigned little endian ints (packs 0, 0 as: 00 00 00 00)
  "V"  -> a 32bit unsigned little endian int (packs 54 as: 36 00 00 00)
```

While I went to the effort of expanding out the byte sequences to make it easier
to see what is going on, you don't typically need to do this at all while
working with `Array#pack` as long as you craft your template strings carefully.
But like anything else in Ruby, it's nice to be able to write little scripts or
hack around a bit in `irb` whenever you're trying to figure out how your
code is actually working.

After figuring out how to encode the file header, the next step was to work on
the DIB header, which includes some metadata about the image and how it should
be displayed on the screen:

```ruby
class BMP 
  class Writer
    DIB_HEADER_SIZE    = 40
    PIXELS_PER_METER   = 2835 # 2835 pixels per meter is basically 72dpi

    # ... other code as before ...

   def write_dib_header(file)
      file << [DIB_HEADER_SIZE, @width, @height, 1, BITS_PER_PIXEL,
               0, pixel_array_size, PIXELS_PER_METER, PIXELS_PER_METER, 
               0, 0].pack("Vl<2v2V2l<2V2")
  end
end
```

Because we are only working on a very limited subset of BMP features, it's
possible to construct the DIB header mostly from preset constants combined with
a few values that we already computed for the BMP file header.

The `pack` statement in the above code works in a very similar fashion as the
code that writes out the BMP file header, with one exception: it needs to handle
signed 32-bit little endian integers. This data type does not have a pattern of its own, 
but instead is a composite pattern made up of two
characters: `l<`. The first character (`l`) instructs Ruby to read a 32-bit
signed integer, and the second character (`<`) tells it to read it in
little-endian byte order.

It isn't clear to me at all why a bitmap image could contain negative values for
its width, height, and pixel density -- this is just how the format is
specified. Because our goal is to learn about binary file processing and not
image format esoterica, it's fine to treat that design decision as a black
box for now and move on to looking at how the pixel array is processed.

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

Sometimes calculations like this are provided for you in format specifications,
other times you need to derive them yourself. Choosing to work
with only 24bit per pixel images allowed me to skirt the question of how to
generalize this computation to an arbitrary amount of bits per pixel.

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

This pattern makes it possible for us to specify color values directly in hexadecimal strings and then convert them to their numeric value just before they get written to the file.

With this last detail explained, you should now understand how to build a
functional bitmap encoder for writing 24bit color images. If seeing things
broken out step by step caused you to lose a sense of the big picture, you can
check out the [source code for BMP::Writer](https://gist.github.com/1351737). Feel free to play around with it a bit before moving on to the next section: the best way to learn is to actually run these code samples and try to extend them and/or break them in various ways.

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

This time, we still are working with an ordinary array of arrays to store the
pixel data, and most of the work gets done as soon as the file is read in the
constructor. Because I decided to support only a single image type, most of the
work of reading the headers is just for validation purposes. In fact, the
`read_bmp_header` method does nothing more than some basic sanity checking, as
shown below:

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

Once these values have been converted into Ruby objects, it's easy to do some
ordinary comparisons to check to see if they're what we'd expect them to be.
Because they help detect corrupted files, clearly defined validations are an
important part of writing any decoder for binary file formats. If you do not do
this sort of sanity checking, you will inevitably run into 
subtle processing errors later on that will be much harder to debug.

As you might expect, the implementation of `read_dib_header` involves more of
the same sort of extractions and validations. It also sets the `@width` and
`@height` variables, which we use later to determine how to traverse the encoded
pixel array.

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
      vres, n_colors, i_colors = header.unpack("Vl<2v2V2l<2V2") 

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

When you take all these code examples and glue them together into a single class
definition, you'll end up with a `BMP::Reader` object that is capable giving you
the width and height of a 24bit BMP image as well as the color of each and every
pixel in the image. For those who'd like to experiment further, the [source code
for BMP::Reader](https://gist.github.com/1352294) is available.

### Reflections

The thing that makes me appreciate binary file formats is that if you just learn
a few basic computing concepts, there are few things that could be more
fundamentally simple to work with. But simple does not necessarily mean easy, and in the process of writing this article I realized that some aspects of binary file processing are not quite as trivial or intuitive as I originally thought they were.

What I can say is that this kind of work gets a whole lot easier with practice.
Due to my work on [Prawn](http://prawnpdf.org) I have written
implementations for various different binary formats including PDF, PNG, JPG,
and TTF. These formats each have their differences, but my experience tells me 
that if you fully understand the examples in this article, then you are already 
well on your way to tackling pretty much any binary file format.

[^1]: To determine the storage space needed for the pixel array in BMP images, I used the computations described in the [Wikipedia article on bitmap images](http://en.wikipedia.org/wiki/BMP_file_format#Pixel_storage).

> NOTE: If you'd like to learn more about this topic, consider doing the Practicing Ruby self-guided course on [Streams, Files, and Sockets](https://practicingruby.com/articles/study-guide-1?u=dc2ab0f9bb). You've already completed one of its reading exercises by working through this article!
