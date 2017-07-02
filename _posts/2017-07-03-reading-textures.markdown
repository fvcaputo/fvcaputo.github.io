---
layout: post
title:  "Some Thoughts on Loading Image Files for Textured Objects"
subtitle: <div style="margin-top:-15px;"><img src="http://i.imgur.com/1FTkgjD.png" style="width:32%;margin-right:0.5%;margin-bottom:5px;" border="1"><img src="http://i.imgur.com/G7elrzr.png" style="width:32%;margin-right:0.5%;margin-bottom:5px;" border="1"><img src="http://i.imgur.com/dBHTKr6.png" style="width:32%;margin-bottom:5px;" border="1">Advancing my OpenGL framework a few steps further by adding object loading and textures, here I'm going to talk a bit about the bumps I had along the way.
date:   2017-07-03 19:51:00
language: en
---

<div style="text-align:center;padding-bottom:10px;">
<a href="http://i.imgur.com/1FTkgjD.png" style="color: black;"><img src="http://i.imgur.com/1FTkgjD.png" style="float: left; width: 32%; margin-right: 0.5%" border="1"></a>
<a href="http://i.imgur.com/G7elrzr.png" style="color: black;"><img src="http://i.imgur.com/G7elrzr.png" style="float: left; width: 32%; margin-right: 0.5%" border="1"></a>
<a href="http://i.imgur.com/dBHTKr6.png" style="color: black;"><img src="http://i.imgur.com/dBHTKr6.png" style="float: left; width: 32%" border="1"></a>
<p><div style="text-align:center;"><font color="gray" size="2px"><p>Some pictures showing the object loader: our good old teapot with flat normals, a textured cube, and a scan of room from Google Tango.</p></font></div></p>
</div>

It's been a while since I talked about my OpenGL framework on this blog (well, almost two years since the last time I talked about it really). But development on it never really ceased, it's the kind of thing that I keep working on even while I'm doing other stuff like the Ray Tracer engine. When I wrote this framework the last time I talked about creating triangle meshes for some primitives and lighting, now I'm going to talk about loading object files and textures.

It'll be a smaller post than the ones I did in the past, mostly because this is something I decided: write more frequently and in more details about small (and interesting steps). I feel that waiting until the whole ray tracing engine is done can be nice to make a big concise post, but at the same time a lot of learning that happened along the way that could be interesting to talk about gets relegated to a couple of sentences.

# Reading Wavefront Files

The file loader that I implemented deals with an old friend of 3D models: Wavefront (.obj) files. This is not only because it's a very common file type to be used for this kind of thing, but also because it's a relatively simple format to read. The file contains vertex data (x, y, z coordinates), normals, texture coordinates and face elements. There is not much mystery in reading this kind of file, processing it is rather easy because the file is very well formatted.

# Reading Image Files

Now, here is the part of reading files that got me thinking about making a post here: reading image files for textures. It was one interesting ride.

# First Lesson About Reading Image Files by Yourself: Don't

Alright, the title for this section is just a joke but one that has meaning behind it. Let me tell you how this "idea" came to be. Right now I'm developing this framework on a Mac, but it actually started back when I was in an older notebook running good old Linux. Back then I already got loading files and loading textures to work, but I did by using a library that I believe a lot of OpenGL devs probably run into at some point in their lives: [SOIL](http://www.lonesock.net/soil.html). This lib makes loading images rather simple, as easy as:

{% highlight c++ %}
textureID = SOIL_load_OGL_texture
    (
        filetexturepath,		
        SOIL_LOAD_AUTO,		
        SOIL_CREATE_NEW_ID,		
        SOIL_FLAG_MIPMAPS | SOIL_FLAG_INVERT_Y		
    );
{% endhighlight %}

Pretty good, right? With the texture ID in place, you can send it to your shaders without much issue. But here is the problem now for me: getting SOIL to work on a Mac. Although it's supposed to be cross platform and work on Mac OS X, I believe this is only for older versions of it (SOIL has not been updated since 2008), and there were plenty of people asking around for help when I googled how to change the build to make it work. After much time wasted on it, and of trying to get some other libs to work instead I decided: screw it, I'm just going to load the image file by myself!

# Reading Image Files: bitmaps

I decided on doing it for a single image format, bitmaps. This was in part because one of the resources that I use to search information on OpenGL is the pretty useful website [opengl-tutorial](http://www.opengl-tutorial.org/), which has a chapter on loading a textured cube by reading an obj file with a bmp texture.

Reading an image file for OpenGL is not as easy as throwing a file pointer to read the file and just dumping that data directly into a `glTexImage2D` call. It requires dealing with header files, knowing what kind of bitmap you have on your hands, what kind of data it has (RGB? RGBA? data described as unsigned bytes?) and where that data actually starts. This I learned by trying to do it. The "problem" with the tutorial on opengl-tutorial about reading bitmaps is that it doesn't go into much detail on this kind of file type, so if you just try to implement the code described on it you will probably run into some problems.

Bitmaps, turns out, have [a lot of versions](http://www.fileformat.info/format/bmp/egff.htm). You cannot just trust that the header will have a fixed number of bytes, because some versions are different from the others. And if you don't know what version you have on your hands, you don't know where the important data is, and you will just end up believing that a certain information is on location, say, `0x0A` when it really isn't.

<div style="text-align:center;padding-bottom:10px;">
<a href="http://i.imgur.com/G7elrzr.png" style="color: black;"><img src="http://i.imgur.com/G7elrzr.png" style="float: left; width: 32%; margin-right: 0.5%" border="1"></a>
<a href="http://i.imgur.com/iup6y6x.png" style="color: black;"><img src="http://i.imgur.com/iup6y6x.png" style="float: left; width: 32%; margin-right: 0.5%" border="1"></a>
<a href="http://i.imgur.com/IsQB2Ai.png" style="color: black;"><img src="http://i.imgur.com/IsQB2Ai.png" style="float: left; width: 32%" border="1"></a>
<p><div style="text-align:center;"><font color="gray" size="2px"><p>How loading your texture can look: the right way, the "reading the image data from the wrong byte location" way and the "reading the wrong pixel format" way.</p></font></div></p>
</div>

To read a file into OpenGL for texture you will need to know at the very least:

* Where the image data starts (not the same as where the header data ends)
* The image's height and width
* The number of color components in the image data (RGB? RGBA?)

This is essentially the data you need to give `glTexImage2D` so the image can be sent to your shaders. Above there are three images showing the same object and the same texture read in different ways (only one correct, as you can see). The left image is the correct cube, with all the data in the correct place. The middle image is what happens if you start reading the image data from the wrong place. You know the total size of the data but you start at the wrong place, either before or after where you should actually start reading, and you get everything shifted). And the right image is what you get when you read the image with the wrong number of components (the file is RGBA and but you set `glTexImage2D` to RGB, everything is bonkers now).

I ran into these problems with bitmaps exactly because of the different formats it has. While I had written a loader that works perfectly with this one bmp file, it suddenly broke with other. That's because the code was not ready to deal with all these formats, and only through some try-and-error approach of changing values I was able to understand why these images were being generated.

# There And Back Again

After some time lost trying to understand bitmap loads file I went back to the original idea of using a library to read an image. But instead if using something more high level as SOIL I went with something true and tried, perfected... you could even say official. I went to [libpng](http://www.libpng.org/pub/png/libpng.html), the "official PNG reference library" which "supports almost all PNG features, is extensible, and has been extensively tested for over 20 years". Beautiful stuff.

The idea here is to take advantage of what I learned I actually need from the image file plus a library that allows me to easily deal with "alien" values. The approach now is different. The png format does not have all these different versions of headers and whatnot that can ruin your loader, it's a much more rigid format. So now it's easy to find the information you want (and the APIs of libpng make it much more easier).

But the real change in the approach comes from the manipulation of the png data. On our final step of loading the image I know the line that will be used is goin to be:

{% highlight c++ %}
glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, width, height, 0, GL_RGBA, GL_UNSIGNED_BYTE, image_data);
{% endhighlight %}

Which means I am only working with RGBA formats that are described by unsigned bytes. The way to deal with that is to change the png internal format. Essentially what is done is, first check how the format of the image you are loading is and change it accordingly. Is it an RGBA format already? Then everything is done. Wait, is it an RGB? Welp, let's fill that data with a white alpha channel. Is the bit depth not 8bit? Change it!

Here is part of that code in action, with API calls from libpng:

{% highlight c++ %}
if(bit_depth == 16)
    png_set_strip_16(pngStruct);

if(color_type == PNG_COLOR_TYPE_PALETTE)
    png_set_palette_to_rgb(pngStruct);

// PNG_COLOR_TYPE_GRAY_ALPHA is always 8 or 16bit depth.
if(color_type == PNG_COLOR_TYPE_GRAY && bit_depth < 8)
    png_set_expand_gray_1_2_4_to_8(pngStruct);

// These color_type don't have an alpha channel then fill it with 0xff.
if(color_type == PNG_COLOR_TYPE_RGB ||
    color_type == PNG_COLOR_TYPE_GRAY ||
    color_type == PNG_COLOR_TYPE_PALETTE)
    png_set_filler(pngStruct, 0xFF, PNG_FILLER_AFTER);
{% endhighlight %}

And then after all of that you are done and can finally send the data to your shader and mess with your objects.

# Some Final Words

That was my small, but full of lessons, journey through the world of image files. I hope this can help people who got stuck in this part of creating your own object loader. I strongly using a third party library to read the data, as it can have all sorts of different formats. Though I to believe a lower level one such as libpng is they way to go if you are interested in learning how this loading actually works, as you can see by this post when I was using SOIL which abstracted all this stuff I had no idea of what it was actually doing (and it was doing A LOT).

Also, can stress this enough, use a very simple object to test your loader. Don't throw in a huge object with tons of geometry and complex texture patters, it will be a lot harder to understand what is going know. You can throw a complicated image (like the room image I used on the header of this post) to test it after you have got your small cube to work.

# References

* [imageHelper.cpp](https://github.com/fvcaputo/openglframework/blob/master/imageHelper.cpp) - My own image loader file, for reference.
* [Opengl-Tutorial](http://www.opengl-tutorial.org/beginners-tutorials/tutorial-5-a-textured-cube/) - Lesson 5: A textured cube.
* [Wikibooks OpenGL Programming](https://en.wikibooks.org/wiki/OpenGL_Programming/Intermediate/Textures#A_simple_libpng_example) - A simple libpng example.
* [Libpng Manual](http://www.libpng.org/pub/png/libpng-manual.txt) - Libpng manual.
