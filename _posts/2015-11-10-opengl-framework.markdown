---
layout: post
title:  "A C++ Framework For Interaction With OpenGL"
subtitle: <div style="margin-top:-15px;padding-bottom:5px;"><img src="http://i.imgur.com/pGCaiQu.png"></div>Let’s talk a little bit about how to make stuff actually show using OpenGL.
date:   2015-11-11 12:07:00
language: en
---
<div style="text-align:center;padding-bottom:10px;">
<img src="http://i.imgur.com/pGCaiQu.png">
<div style="text-align:center;"><font color="gray" size="2px">A few things you can create using the framework tools.</font></div>
</div>

Hello people. So I guess there is no better place to start than from the beginning, huh? So here we go, first actual post here will be about this project that I have been working on (and tweaking) for a while now, a framework for interaction with OpenGL.

What I have been doing for a while is implementing this framework from the ground up. I wanted to get a good feel of the graphics pipeline by building every small part of it. That means, make something that doesn’t use libraries such as OpenGL Mathematics, not that there is anything wrong with those, but rather I want to implement everything. So let’s get down to what is currently available in the framework.

<div style="text-align:center;padding-bottom:10px;">
<iframe width="560" height="315" src="https://www.youtube.com/embed/NTnnJLC7ujc" frameborder="0" allowfullscreen></iframe>
<div style="text-align:center;margin-top:-5px;"><font color="gray" size="2px">Here's a video showing pretty much all that I have on the framework: camera controls, different shapes, light model, etc.</font></div>
</div>

# The Stuff

The plan was to create the framework in such a way that would be easy to use in other environments and to improve upon. After some time working on the design I ended up implementing the following:

* A Vertex and Fragment shader reader.
* A Shape creator with a few primitives such as cube, cylinder and sphere.
* Some simple transformations (translation, rotation, scaling).
* Phong Illumination.
* Camera manipulation.

All of it was implemented in C++ taking advantage of object oriented programming. For instance, a shape is literally a class that I instantiate for different objects that I want, and then I can get the geometry of those shapes through functions.

# Reading Shaders

One thing that took a while was to make my own shader reader. In other words, the code necessary to read the OpenGL Shading Language (GLSL), compile it and link it to a “program”. I implemented it in such a way that logs would be displayed if any errors happened when compiling the code (and man, that saved me a lot of headaches). So far it works for Vertex and Fragment shader, I’ll be working on geometry shaders later.

{% highlight c++ %}
// Load shaders
GLuint program;
program = shader::makeShaderProgram( "shaders/simpleVert.glsl",
                                     "shaders/simpleFrag.glsl" );
{% endhighlight %}
<div style="text-align:center;margin-top:-15px;"><font color="gray" size="2px">Reading shaders and linking to a program.</font></div>

# I Need Shapes!

<div style="text-align:center;padding-bottom:10px;">
<iframe width="560" height="315" src="https://www.youtube.com/embed/Ig41MueDxBE" frameborder="0" allowfullscreen></iframe>
<div style="text-align:center;margin-top:-5px;"><font color="gray" size="2">Showing the triangle mesh of different shapes, and chaging the subdivision on the fly.</font></div>
</div>

Next on the list was the geometry: actually making shapes. For starters we have three primitives: a cube, a cylinder and a sphere. I’m able to manipulate the mesh of these shapes by increasing the number of subdivisions for each one.

The cube is simple, each square made of two triangles, and each side of the cube is divided into rows and columns of those squares, depending on how many you want. The cylinder has different subdivisions for both the base and the height. The height is made of simple squares, and the base was created using polar coordinates. And finally the sphere, which is made from icosahedron subdivisions.

{% highlight c++ %}
// Creating a cube, three sub divisions
Shape shape;
shape.makeCube(3);

// setting materials
shape.setMaterials(0.5f, 0.1f, 0.9f, 0.5f,
                   0.89f, 0.0f, 0.0f, 0.7f,
                   1.0f, 1.0f, 1.0f, 1.0f, 10.0f);

shape.getVertices(); // return vertices
{% endhighlight %}
<div style="text-align:center;margin-top:-15px;padding-bottom:10px;"><font color="gray" size="2px">Really easy to create shapes.</font></div>

The geometry for the shapes also generate normals, which can be either “flat” or “smooth”. Those are important for a number of applications, like illumination. And speaking of illumination, each shape can also have materials assigned so it can be illuminated by the phong illumination model.

# Moving

Not that fancy really, the basic transformations you need for computer graphics is here: translation, scale and rotation (in Euler angles, no fancy quaternions yet).

{% highlight c++ %}
// Creating a transformation matrix with some rotations and a translation
Matrix mTransform = translate(-1,-0.9,-4) * rotate(ztheta, zVec) * rotate(ytheta, yVec) * rotate(xtheta, xVec);
{% endhighlight %}
<div style="text-align:center;margin-top:-15px;padding-bottom:10px;"><font color="gray" size="2px">No mistery here.</font></div>

# Around The World, Around The World

Well having those shapes and being able to move stuff is not fun if I can’t look at them in different angles: enter the camera. The pipeline is not there if we can’t generate the good old View Matrix and Projection Matrix, so let’s do it.

On the framework you can set up your projection however you like, with orthogonal or perspective projections. And also, you can move around the camera (or rather the world, but bear with me for the sake of simplicity) to look at your world however you prefer, using the familiar WASD keys and/or the mouse (hold the left mouse key and move around).

{% highlight c++ %}
// Our Camera, perspective projection
Camera cam(PROJ_PERSP);

// gets the view matrix
cam.getViewMatrix();

// gets the projection matrix
cam.getProjMatrix();
{% endhighlight %}
<div style="text-align:center;margin-top:-15px;padding-bottom:10px;"><font color="gray" size="2px">Create a camera with a certain projection and get the matrices you need.</font></div>

# Light up the night

<div style="text-align:center;padding-bottom:10px;">
<iframe width="560" height="315" src="https://www.youtube.com/embed/Ev51Wc0JwCA" frameborder="0" allowfullscreen></iframe>
<div style="text-align:center;margin-top:-5px;"><font color="gray" size="2">Flat, gouraud and phong shading on a sphere, plus different materials.</font></div>
</div>

Finally, you don’t wat to look just at wire frames, how about some solid objects? I have got you covered, we have the Phong Illumination model here waiting for you. Like I mentioned before, you are able to assign materials for any shape you want to, and those values will be used when you set up the light in the world. By implementing the correct shaders you can use an object’s material and normals, together with the light source to make the object visible however you prefer.

{% highlight c++ %}
// Creating a light source,
Lighting light(2.0f, 2.0f, -4.0f,  // the light position (x, y, z)
               1.0f, 1.0f,  1.0f,  // the light intensity (r, g, b)
               0.5f, 0.5f,  0.5f); // the value of the ambient light (r, g, b)

// set up phong for a shape
light.setPhongIllumination(program, cube);
{% endhighlight %}
<div style="text-align:center;margin-top:-15px;padding-bottom:10px;"><font color="gray" size="2px">Just create a light source and set it up for a shape.</font></div>

# That’s All (For Now)

And that is what I have got so far. It was fun implementing all this stuff and seeing it work. And now with this framework I have got the basics that I need to make fancy stuff. More illumination models, raymarching, texture mapping… let’s see what happens next.

All this code is up on my github. Not only the framework itself, but also the code for the three videos you can watch here (or in my youtube channel) is out there.
