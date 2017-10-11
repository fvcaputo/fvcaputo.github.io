---
layout: page
title: Portfolio
permalink: /portfolio/
---

# Ray Tracer

A Ray Tracer engine written from the ground up in C++.

#### Features

- Space partitioning data structure (k-d tree).
- CPU Multithreading.
- Texture Mapping support (both procedural and through image loading).
- Model loading of .ply files.
- Area lighting.
- Volumetric Lighting through ray marching.

#### Related Posts

- [Implementing a Ray Tracer]({% post_url 2017-04-16-ray-tracer %})
- [Creating Volumetric Lights and Shadows Through Ray Marching]({% post_url 2017-05-02-ray-marching %})

#### Resources

The complete code is available at my [GitHub Repo](https://github.com/fvcaputo/raytracer).
<br>
<div align="center">
<a href="http://i.imgur.com/QKPD53g.png" style="color: black;"><img src="http://i.imgur.com/QKPD53g.png" style="width: 32%; margin-right: 0.5%"></a>
<a href="http://i.imgur.com/uGuqSOk.png" style="color: black;"><img src="http://i.imgur.com/uGuqSOk.png" style="width: 32%; margin-right: 0.5%"></a>
<a href="http://i.imgur.com/fKUDnof.png" style="color: black;"><img src="http://i.imgur.com/fKUDnof.png" style="width: 32%"></a>
</div>
<div align="center" style="padding-top:10px">
<a href="http://i.imgur.com/RjSN5Gr.png" style="color: black;"><img src="http://i.imgur.com/RjSN5Gr.png" style="width: 32%; margin-right: 0.5%"></a>
<a href="http://i.imgur.com/fcwHYPB.png" style="color: black;"><img src="http://i.imgur.com/fcwHYPB.png" style="width: 32%; margin-right: 0.5%"></a>
<a href="http://i.imgur.com/7a99qgm.png" style="color: black;"><img src="http://i.imgur.com/7a99qgm.png" style=" width: 32%"></a>
</div>
<br>

--------------------------------------------------------------------------------
<br>
# Virtual Theatre

This project is diveded intwo applications, one is an application built using Unity and Vuforia to display Motion Capture data in an Augmented Reality experience and the other also built in Unity uses Microsoft's plugin to capture motion data.

#### Features

- Uses Vuforia SDK to display AR graphics through target recognition.
- Can be deployed to any android smartphone/tablet device.
- Uses the Moverio SDK to be able to run on the Epson Moverio BT-200 glasses.
- Animation through Keyframing and TCB interpolation.
- The application for motion tracking uses Microsoft's Kinect plugin to track the data.

#### Related Posts

- [Using Motion Capture to Animate Virtual Characters in a Physical Stage]({% post_url 2016-06-23-augmented-reality %})

#### Resources

The code for this project is separated into two repositories, the one to build the AR application ([here](https://github.com/fvcaputo/virtualtheatre)) and the one for motion capturing ([here](https://github.com/fvcaputo/kinectbodytracking)).

#### Notes

I developed this project during grad school, it was something that was developed in conjuntction to another project called Farewell to Dawn that was accepeted and presented as a poster project at [SIGGRAPH '16](https://dl.acm.org/citation.cfm?id=2945127&CFID=817523326&CFTOKEN=26501141). A video of it can be seen on the ACM page under "Source Materials".
<br>
<div style="text-align:center;padding-bottom:10px;">
<iframe width="640" height="360" src="https://www.youtube.com/embed/Hmh5L7BBmJM" frameborder="0" allowfullscreen></iframe>
</div>

--------------------------------------------------------------------------------
<br>
# OpenGL Framework

An OpenGL framework written from the ground up in C++ with a few helpful external libs. :)

#### Features

- Custom vertex and fragment shader reader.
- Custom model loading of Wavefront files.
- Primitives creator with mesh subdivision control (cube, spheres, cylinders).
- Phong Illumination lighting.
- Light mapping support (diffuse mapping, specular mapping, normal mapping).


#### Related Posts

- [A C++ Framework For Interaction With OpenGL]({% post_url 2015-11-10-opengl-framework %})
- [Some Thoughts on Loading Image Files for Textured Objects]({% post_url 2017-07-03-reading-textures %})

#### Resources

The complete code is available at my [GitHub Repo](https://github.com/fvcaputo/openglframework).
<br>
<div align="center">
<a href="http://i.imgur.com/1FTkgjD.png" style="color: black;"><img src="http://i.imgur.com/1FTkgjD.png" style="width: 32%; margin-right: 0.5%"></a>
<a href="http://i.imgur.com/G7elrzr.png" style="color: black;"><img src="http://i.imgur.com/G7elrzr.png" style="width: 32%; margin-right: 0.5%"></a>
<a href="http://i.imgur.com/dBHTKr6.png" style="color: black;"><img src="http://i.imgur.com/dBHTKr6.png" style="width: 32%"></a>
</div>
<br>
--------------------------------------------------------------------------------
<br>
# Sand Particle Simulation

A simulation of the behaviout of sand particles built using in Unity as a framework.

#### Features

- Custom Particle System (emitter, position update, etc).
- Custom collision detection.
- Colission behaviour modelled after sand behaviour.
- Animation of models though Runge-Kutta integration.

#### Related Posts

- [Simulating Sand Using Particle Systems]({% post_url 2016-01-11-sand-particle-system-in-unity %})

#### Resources

The complete code is available at my [GitHub Repo](https://github.com/fvcaputo/sandparticle).
<br>
<div style="text-align:center;padding-bottom:10px;">
<iframe width="640" height="360" src="https://www.youtube.com/embed/auhUHaOxI8o" frameborder="0" allowfullscreen></iframe>
</div>
