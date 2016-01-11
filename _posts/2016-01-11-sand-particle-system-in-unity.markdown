---
layout: post
title:  "Simulating Sand Using Particle Systems"
subtitle: <div style="text-align:center;margin-top:-15px;padding-bottom:5px;"><img src="/assets/2016-01-11-sand-particle-system-in-unity/fallingSandObstables1.png" style="width:33%;margin-right:0.5%"><img src="/assets/2016-01-11-sand-particle-system-in-unity/fallingSandObstables2.png" style="width:33%;margin-right:0.5%"><img src="/assets/2016-01-11-sand-particle-system-in-unity/fallingSandObstables3.png" style="width:33%;"></div>Particle systems are fun by itself and we can use it to simulate all sorts of different things, so I decided to use one to simulate sand. This time using Unity as a framework.
date:   2016-01-11 12:00:00
language: en
---

<div style="text-align:center;padding-bottom:10px;">
<img src="/assets/2016-01-11-sand-particle-system-in-unity/fallingSandObstables1.png" style="float: left; width: 33%; margin-right: 0.5%">
<img src="/assets/2016-01-11-sand-particle-system-in-unity/fallingSandObstables2.png" style="float: left; width: 33%; margin-right: 0.5%">
<img src="/assets/2016-01-11-sand-particle-system-in-unity/fallingSandObstables3.png" style="float: left; width: 33%">
<div style="text-align:center;"><font color="gray" size="2px">Falling sand on a few obstacles.</font></div>
</div>

Alright so let's get down to it. For this little project I decided to try my hand at the Unity engine. However not in the usual way, instead of using all the wonderful tools that are already in the engine (like its built-in particle system or its physics engine) I implemented all that stuff myself. Essentially I used Unity as you would use a bare bones graphics engine. I used it to create objects, produce lighting, manipulate the camera, update frames, and some built-in functions for transformations and the like.

I got a pretty decent result in the end, with some really believable sand motion. You can check below a video showing four different scenes I came up with to showcase the system in action.

<div style="text-align:center;padding-bottom:10px;">
<iframe width="560" height="315" src="https://www.youtube.com/embed/auhUHaOxI8o" frameborder="0" allowfullscreen></iframe>
<div style="text-align:center;margin-top:-5px;"><font color="gray" size="2px">Four different scenes showing my sand simulation.</font></div>
</div>

Now let's go into more detail on the whole thing.

# The Particle System

As I mentioned before, instead of using the built-in tools Unity provides us with, I implemented pretty much everything myself. So the first thing on the list was a particle system.

The particle system itself can be used as a mean to model different phenomena such as fire, water, smoke and in our case, granular materials. But regardless of what you are trying to simulate, if you are building it by using a particle system then it will pretty much be the same always. 

I use a basic system where I have a particle emitter, update functions and, of course, the particles. 

# The Particle Emitter

The emitter is set up with a number of parameters that can be changed depending on what exactly you want. And they are:

* **Position** - A (x, y, z) vector representing the position of the emitter.
* **Yaw** and **Pitch** - Values that affect the force direction the particles are going to have when they are created.
* **Total Particles** - The total number of particles the emitter will generate.
* **Emission Rate** - How many seconds to wait to generate new particles.
* **Emission Per Rate** - How many particles will be generated in every emission.
* **Lifetime** - How long each particle will stay alive.
* **Variances** - For almost all values above we add a variance, which is a value to randomize the emission.

So the particle emitter is really malleable and we can use it in all sorts of different ways. We can say where the particles are going to created and how they are going to be moving. We can add variance to those values if we want to, if we don't what the emission of every single particle to be the same. We can play around with all of that to get some really cool results.

The particle emitter itself is a really simple class, and it's generic in the sense that it can be used for any sort of particle. If you check the code you will see:
{% highlight csharp %}
public class ParticleEmitter {
    // Our Particle
    public SandParticle[] particles;
    ...
    // Constructor
    public ParticleEmitter(Vector3 newPos, Vector3 newPosVar, float newYaw, float newYawVar, float newPitch, float newPitchVar, 
        int newTotalParticles, float newEmissionRateSec, int newEmitsPerRate, float newLife) { ... }
    ...
    // Emit will return a particle list, which will be rendered
    public void updateParticles(float dt) { ... }
}
{% endhighlight %}
Above you can see just a few parts of the code I'm going to mention. The emitter needs to have a kind of particle inside of it, in this case it's a class called "SandParticle". Again, it could be something else, provided you made the necessary changes. Then constructor would be changed if you wanted other values, and _updateParticles_ would call the constructor of the particles you are using and the update function that particle class should have.


# The Particles and How They Move

Well, here is where the specifics of the system actually start. After an emitter is created then we actually need to start it. To do that we need to update the values of every single particle, and of course this depends on how our particles look and how we want them to move.

So first let's talk about our particles. What I want is to simulate sand, therefore those particles will need to be affected by physics. For this system every particle will be a sphere that will have a certain mass and radius. Each particle will have velocity, momentum and force for both translational and rotation motion. Particles are also affected by external foces such as gravity and friction. Since I am modeling it in such a way that all particles are spheres we can also do some approximations to facilitate calculations, such as the inertia tensor used to calculate the angular velocity. This is pretty much the standard you would expect from any physics engine. What really creates the believable movement of sand is the behavior of the particles during collisions with other objects.

Checking the code the SandParticle class has these main functions:
{% highlight csharp %}
// Constructor of a particle, given a position and a start force. It's called by the emitter
public SandParticle(Vector3 startPosition, Vector3 startForce) { ... }
...
// This is how we update this particle. We need a time step dt, which will be used in the integration
// and all the other particles alive to check for collisions
public void updateParticle(float dt, SandParticle[] otherParticles, int aliveParticles) { ... }
{% endhighlight %}
I will be talking more about updating the particle and the collisions below. These functions should always be called my the emitter, which in Unity should be instantiated on the script that will actually run during the scene.

#### Collisions

<div style="text-align:center;padding-bottom:10px;">
<img src="/assets/2016-01-11-sand-particle-system-in-unity/explodingSand1.png" style="float: left; width: 33%; margin-right: 0.5%">
<img src="/assets/2016-01-11-sand-particle-system-in-unity/explodingSand2.png" style="float: left; width: 33%; margin-right: 0.5%">
<img src="/assets/2016-01-11-sand-particle-system-in-unity/explodingSand3.png" style="float: left; width: 33%">
<div style="text-align:center;"><font color="gray" size="2px">Particles created in a small space "explode" because they are penetrating each other. The simulation runs until it reaches an equilibrium.</font></div>
</div>

The collision detection done in the simulation is done by checking each pair of objects. It's the naive approach with time complexity of $$\mathcal{O}(n^{2})$$, where n is the number of objects on the scene, which definitely not the optimal way of doing this. But since I am not dealing with a ridiculously large number of particles this suffices, the rendering of the scene is slow but good enough. And the detection between particles is simple, since I'm dealing only with spheres we just need to check the distance between the center of mass of each sphere. Same thing when I am dealing with "planes" (floors and walls).

Collision response is where the behavior of sand is actually implemented. When a pair of particles collide we calculate a contact force based on Hooke's Law, essentially using invisible springs between the particles that applies a contact force on the particles. This contact force is composed of normal term $$F_{n}$$ and shear term $$F_{s}$$ which are calculated using the the following equations:

$$
\begin{equation*}
\begin{split}
F_{n} & = - (k_{n}\delta^{3/2} + \gamma_{n}\dot{\delta}\delta^{1/2}) n \\ 
F_{t} & = - min (\mu||F_{n}||,k_{s}||v_{t}||) \frac{v_{t}}{||v_{t}||}
\end{split}
\end{equation*}
$$

Now, if you want to learn the specifics of those equations check the references section of this post, here I'm only going to talk briefly about some variables. $$k_{n}$$ and $$\gamma_{n}$$ are the stiffness and damping coefficient, $$\mu$$ and $$k_{s}$$ are the Coulomb and viscous friction coefficients. Those four coefficients are constants that should have different values for different behaviors. From the articles in the references you can see some ranges of values that were used, and that's because those values come from experimenting. To find the values that I personally thought were the most believable ones I did a handful of tests with different values. And in fact, every collision on my simulation uses those equations, which means collisions between sand particles and the floor use it, as well as collisions between the particles and the other spheres. But for different cases those values are also different.

#### Updating Particles

To update the simulation Runge-Kutta integration was used. It's not the only method that can be used in such a simulation, but it gives us good enough results given a small enough timestep. Like I mentioned before the system needs a way to update the values of every particle, which needs to be called every time a new frame is rendered. Runge-Kutta will need a value _dt_ to update the particles. And this value is very important because depending on it the error in the calculations might be too large and the movement will look completely wrong. 

You saw above some scenes with a time step of 3 ms. Now check how the first scene would look if you had instead 6 ms.

<div style="text-align:center;padding-bottom:10px;">
<iframe width="560" height="315" src="https://www.youtube.com/embed/44PSBoqFJOg" frameborder="0" allowfullscreen></iframe>
<div style="text-align:center;margin-top:-5px;"><font color="gray" size="2px">Using a larger time step in the integration can give us some weird results.</font></div>
</div>

Funky, right? The only change in the scene is the timestep of the integration, every other value was kept the same.

# Cheating: Dealing with Different Collisions

<div style="text-align:center;padding-bottom:10px;">
<img src="/assets/2016-01-11-sand-particle-system-in-unity/fallingSandBody1.png" style="float: left; width: 33%; margin-right: 0.5%">
<img src="/assets/2016-01-11-sand-particle-system-in-unity/fallingSandBody2.png" style="float: left; width: 33%; margin-right: 0.5%">
<img src="/assets/2016-01-11-sand-particle-system-in-unity/fallingSandBody3.png" style="float: left; width: 33%">
<div style="text-align:center;"><font color="gray" size="2px">Sand falling on a body.</font></div>
</div>

So I talked before about how collisions between spheres is really easy to detect. However, you also saw on the videos and on the pictures that other types of collisions are happening. Those are are collisions between the spheres and the floor/wall, and even between a human-like body made out of boxes. How does that work?

Well, dealing with planes is simple. For instance, you know where the floor is beforehand (let's say it's y = 0), so you can just check the collision between the center of mass of the spheres and that plane, which is trivial. But the collision of spheres and boxes (what looks like I'm doing with the human body) is a whole different thing. That's why what I did to simplify this step was "cheat" and actually create invisible spheres that are placed in the same positions as the boxes that make the body, as you can see on the picture below. That gives a good enough result without having to implement a whole different method of collision detection to deal with that.

<div style="text-align:center;padding-bottom:10px;">
<img src="/assets/2016-01-11-sand-particle-system-in-unity/realBody.png" style="width:40%;">
<div style="text-align:center;"><font color="gray" size="2px">This is what you are not seeing.</font></div>
</div>

# Results

The results, I'd say, where really good. It's possible to see the expected motion sand would have, such as piling up and moving with a sort of "viscous" behavior. The rendering takes a while depending on the number of particles you are using, mostly because of the way I'm detecting collisions. That can be optimized using a space-partitioning data structure to organize the space, which I might add at a later time. But overall, I'm really please with the results.

# References

If you are interested in learning more about all this stuff, here are the references I used.

* [Unity][unity] - Unity's website
* [Gaffer on Games][integration] - Good article to get a general idea of integration basics.
* [The Ocean Spray in Your Face][ocean] - Great article about partycle systems.
* Ivn Aldun, Angel Tena, and Miguel A. Otaduy, _Simulation of High-Resolution Granular Media_, CEIG09, San Sebastin, Sept. 9-11 (2009).
* Nathan Bell, Yizhou Yu and Peter J. Mucha, _Particle-Based Simulation of Granular Materials_, Eurographics/ACM SIGGRAPH Symposium on Computer Animation (2005).

[unity]: https://unity3d.com/
[integration]: http://gafferongames.com/game-physics/integration-basics/
[ocean]: https://www.lri.fr/~mbl/ENS/IG2/devoir2/files/docs/particles.pdf
