Escper (read: "escaper") -- Easy printing to serial thermal printers
=======================================================================

Installation
------------

    gem install escper
    
Introduction
------------

Escper is a collection of essential tools that make printing of plain text and images to one or many serial thermal printers easy. Both USB and serial (RS232) printers are supported and detected automatically. Escper is useful for Ruby based Point of Sale systems that want to print receipts or tickets.

While the actual printing is just writing a string to a device node inside of `/dev`, there is some preprocessing necessary. Thermal printers usually have a narrow character set, do not support UTF-8, only ASCII-8BIT. Escper makes it possible to just send UTF-8 text and it does all the conversion work under the covers. For special characters, Escper will map UTF-8 characters to the ASCII-8BIT codes that are actually supported by the currently set codepage on the printer (have a look at the user manual of your printer). The file `lib/escper/codepages.yml` is an example for matching UTF-8 codes to ASCII codes.

Another tricky usecase is where text is mixed with images. Images are transformed into ESCPOS compatible printing data, which is supported by the majority of thermal printers. When mixing text and images, Escper will transform special UTF-8 characters according to the above mentioned codepages.yml file, but leave the image data unchanged.

Some Point of Sale systems, especially those for restaurants, have to send tickets to several printers at the same time (e.g. kitchen, bar, etc.). Escper makes this easy. Have a look at the usage examples below.

Usecase: Convert image to ESCPOS code
------------

Load Escper:

    require 'escper'
    
The source of the image can be a file:

    escpos_code = Escper::Img.new('/path/to/test.png', :file).to_s

The source of the image can also be data which is uploaded from a HTML form:

    escpos_code = Escper::Img.new(data.read, :blob).to_s
    
"Data" is a variable containing the image data of a multipart HTML form.
    
Alternatively, the source can be an ImageMagick canvas:

    canvas = Magick::Image.new(512, 128)
    gc = Magick::Draw.new
    gc.stroke('black')
    gc.stroke_width(5)
    gc.fill('white')
    gc.fill_opacity(0)
    gc.stroke_antialias(false)
    gc.stroke_linejoin('round')
    gc.translate(-10,-39)
    gc.scale(1.11,0.68)
    gc.path("M 14,57 L 14,56 L 15,58 L 13,58 L 14,57 L 21,59 28,60 34,62 40,65 46,67 52,68 56,70 61,72 66,73 70,74 75,74")
    gc.draw(canvas)
    escpos_code = Escper::Img.new(canvas,:obj).to_s

For optimal visual results, when using a file or a blob, the image should previously be converted to an indexed, black and white 1-bit palette image. In Gimp, click on "Image -> Mode -> Indexed..." and select "Use black and white (1-bit) palette". For dithering, choose "Floyd-Steinberg (reduced color bleeding)". The image size depends on the resolution of the printer. The Escper gem contains a test image in `lib/escper/test.png`.

To send an image directly to a thermal receipt printer in just one line:

    File.open('/dev/usb/lp0','w') { |f| f.write Escper::Img.new('/path/to/image.png').to_s }
    
Usecase: Print UTF-8 text to several serial printers at the same time
------------

First, create printer objects which are simply containers for data about the printer. If you develop your application in a framework like Rails, you also can use the models from this application instead of creating them from within Escper. The only requirements are that the objects respond to the methods `id`, `name`, `path`, `copies` and `codepage`.

    vp1 = Escper::VendorPrinter.new :id => 1, :name => 'Printer 1 USB', :path => '/dev/usb/lp0', :copies => 1
    vp2 = Escper::VendorPrinter.new :id => 2, :name => 'Printer 2 USB', :path => '/dev/usb/lp0', :copies => 1
    vp3 = Escper::VendorPrinter.new :id => 3, :name => 'Printer 3 RS232', :path => '/dev/ttyUSB0', :copies => 1
    
`id` must be unique numbers which will be needed later during printing. `name` is an arbitrary string which is only used for logging. `path` is the path to a regular file or the device node of the thermal printer which usually lives under `/dev`. Device nodes can be of the type USB or serial (RS232). `copies` are the number of copies of pages that the printer should print. Optionally, you can pass in the key `codepage` which is a number that must correspond to one of the YAML keys in the file `codepages.yml`. By making the `codepage` a parameter of the printer model, it is possible to use several different printers from different manufacturers with different character sets at the same time.

Next, initialize the printing engine of Escper by passing it an array of the printer objects. It is also possible to pass it a single printer model instead of an array:

    print_engine = Escper::Printer.new 'local', [vp1, vp2, vp3]
    
Now, open all device nodes as specified in `new`:

    print_engine.open
    
Once all device nodes are openend for writing, you can print text. Text must be UTF-8 encoded. The fist parameter is the `id` of the printer object that should be printed to. The second parameter, the text, will be printed immediately:

    print_engine.print 1, 'print text to printer 1'
    print_engine.print 2, 'print text to printer 1'
    print_engine.print 3, 'print text to printer 1'
    
The `print` method will return an array which contains the number of bytes actually written to the device node, as well as the raw text that was written to the device node.
    
After printing is finished, don't forget to close the device nodes which were openened during the `open` call:

    print_engine.close

    
Usecase: Print UTF8 text mixed with binary images
------------

Create a printer object:

    vp1 = Escper::VendorPrinter.new :id => 1, :name => 'Printer 1 USB', :path => '/dev/usb/lp0', :copies => 1
    
Render images into ESCPOS, stored in variables:

    image_escpos1 = Escper::Img.new('/path/to/test1.png', :file).to_s
    image_escpos2 = Escper::Img.new('/path/to/test2.png', :file).to_s
    
This raw image data now must be stored in a hash, whith a unique key that gives the image a unique name:

    raw_images = {:image1 => image_escpos1, :image2 => image_escpos2}
    
Initialize the print engine:

    print_engine = Escper::Printer.new 'local', vp1
    
Open the printer device nodes:

    print_engine.open
    
Print text and image at the same time:

    print_engine.print 1, 'print text and image1 {::escper}image1{:/} and image 2 {::escper}image1{:/} to printer 1', raw_images
    
Note that the magic tag `{::escper}image_key{:/}` will be replaced by Escper with the image that is stored in the hash `raw_images` with the key `image_key`.

After printing is done, close the device nodes again:

    print_engine.close
    

Fallback mode
----------------------

If a device node is busy or not writable, Escper will create fallback files instead and append the print data to that file. The fallback files will have the name of the printers and will be saved inside of `/tmp`, or, when you include Escper from Rails, in the `tmp` folder of the Rails app source.

Configuration
----------------------

You can configure Escper by calling in your project:

    Escper.setup do |config|
      config.codepage_file = File.join('path', 'to', 'codepages.yml')
      config.use_safe_device_path = false
      config.safe_device_path = File.join('path', 'to', 'outputdir')
    end

`codepage_file` specifies the path to `codepages.yml`. (for an explanation see above)

`use_safe_device_path` can be set to `true` when you are running Escper on a remote server and no actual writes to device nodes should occur. In this case, all print data will be stored in regular files in the path `safe_device_path` with safe file names, which can be further processed or submitted by other programs.
    
Additional Features
----------------------

For additional features, please study the source code.


Contact
----------------------

Ask for support or additional features!

Red (E) Tools Ltd.

office@red-e.eu

www.red-e.eu


Licence
----------------------

Copyright (C) 2011-2012  Michael Franzl <michael@billgastro.com>

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU Affero General Public License as
published by the Free Software Foundation, either version 3 of the
License, or (at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU Affero General Public License for more details.

You should have received a copy of the GNU Affero General Public License
along with this program.  If not, see <http://www.gnu.org/licenses/>.
