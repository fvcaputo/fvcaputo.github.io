---
layout: post
title:  "Using Motion Capture to Animate Virtual Characters in a Physical Stage"
subtitle: <div style="margin-top:-15px;padding-bottom:5px;"><img src="http://i.imgur.com/59plKev.png" style="width:49%;margin-right:0.5%"><img src="http://i.imgur.com/A3doLfK.png" style="width:49%"></div>This time let's play a bit with the Microsoft Kinect, Unity and use Vuforia to create an augmented reality application for mobile devices to create animations of virtual characters.
date:   2016-06-23 12:00:00
language: en
---

<div style="text-align:center;padding-bottom:10px;">
<img src="http://i.imgur.com/59plKev.png" style="float: left; width: 49%; margin-right: 0.5%">
<img src="http://i.imgur.com/A3doLfK.png" style="float: left; width: 49%">
<div style="text-align:center;"><font color="gray" size="2px">Screenshots of the AR application showing the same model in different angles.</font></div>
</div>

This time I was interested in learning to develop augmented reality applications, which got me looking into the Vuforia SDK. And looking at the tools I had in my disposal just laying around, this project took a far greater scope that I had in mind, and I have only my own crazy ideas to blame.

Yes, the main thing was still getting something interesting to show in AR, but what would that be? I decided to use motion capture, using a Micrososft Kinect v2, to track the motion of a person and use that data to animate a virtual character, which is ultimately is displayed in my AR application (and in a VR application I ended up developing aswell).

Below I'll go over the three main aspects of this whole system, and I'll show the final results that I got with the application running on an Android smartphone.

# The Motion Capture

Motion capture is the process of recording movement, of either objects or people, and transforming it into a mathematical representation that can be used to recreate the movement. This process is based on tracking key points in the subject’s movement.

As mentioned before, I used a Micosoft Kinect v2 for this step. The Kinect is able to extract what it calls "skeletal data", which is the three dimensional position of 25 joints in the human body. This is the data used to animate the virtual character, which is a process called Performance Animation.

The Kinect data is not completely reliable, it can actually be quite noisy and it has quite a limited range to track the subject's movement. When it detects a person, it'll start tracking the joints of that person with different degrees of confidentece (not tracked, tracked and inferred), in a small field of view. In fact it will most likely completely lose track of the person if that person is sideways to the device. So obviously it's not the kind of thing you would use for a serious motion capture session, such as the one used in movies, but for my purposes it's quite enough.

<div style="text-align:center;padding-bottom:10px;">
<img src="http://i.imgur.com/0thJbuC.png" style="float: left; width: 49%; margin-right: 0.5%">
<img src="http://i.imgur.com/fXsC8so.png" style="float: left; width: 49%">
<div style="text-align:center;"><font color="gray" size="2px">Result of the motion capture on the Unity application.</font></div>
</div>

I built an application in Unity, which used Micosoft's plugin for the Kinect to be able to extract the joint data. This application runs on a PC which has a Kinect connected to it, and is able to either store the kinect data into a file or transmit that data in real time. This is used by the AR (and VR) application running on a mobile device to render the virtual character.

# The Virtual Character

With the positions extracted from the motion capture I'm able to animate a simple virtual character. Not only I have the three dimensional positions of every joint, but because the Kinect also tags each joint, I also know which position corresponds to which joint (I'm able to tell which of the 25 joints is the head joint, the right knee, the hips, etc). And with that information it's possible to map those positions to a virtual model.

I'm not particularly good at modeling at all, so the models I created were really simple, just to give an idea of what is possible to do with that sort of data. We can create simple stick figures with lines connecting the joints, we can place blocks between the joints, and even use particle systems based on the joints position.

<div style="text-align:center;padding-bottom:10px;">
<img src="http://i.imgur.com/TIiAKyV.png" style="float: left; width: 33%; margin-right: 0.5%">
<img src="http://i.imgur.com/B3Qfz5I.png" style="float: left; width: 33%; margin-right: 0.5%">
<img src="http://i.imgur.com/ajn5gKx.png" style="float: left; width: 33%">
<div style="text-align:center;"><font color="gray" size="2px">Three virtual character models created as examples.</font></div>
</div>

# Animation

So I have the data from the Kinect, and the models for the virtual character, but now I actually need to animate it. When considering computer animation techniques we have three main approaches, techniques based on interpolation (e.g. keyframing animation), data driven animation (e.g. performance based animation) and procedural animation. For the purposes of this project, I'm are interested in keyframing animation.

Keyframing animation is a technique based on interpolation. It is based on defining and creating the key frames of the sequence to be animated. In traditional animation, key frames would be drawn by the animators and all the intermediate frames between them would be made by the assistants. Those frames are then displayed in rapid succession to create the illusion of movement. In computer animation the process is essentially the same, with the difference that after defining the key frames, the intermediate frames can be calculated by the computer. Key frames consist of certain variables established by the animator, for instance values for position and orientation, and for every frame generated between two consecutive key frames those variables are interpolated using the key frame values as the extremes.

This process is also used so that the animation becomes independent of the computer’s processing power, because each key frame is defined based on time. The idea is to define which key frame is redered at which specific time during the animation, so that any device running the process will show the same animation independently of how efficiently it can interpolate the data.

To animate the character I have implemented both linear interpolation and TCB interpolation. The approach I took was to create a Keyframe class, called `KeyFrameAnimation`, which should be created for each joint of the virtual model. So you would have an instance of the class for the head joint alone, and one for each of the others. And in each keyframe class instance, you are going to add all the positions for each keyframe the joint will have (based on the motion capture joint data).

{% highlight csharp %}
// For the key frame, each joint needs a key frame object
public class KinectJoint
{
    // joint name
    public string name;

    // every joint needs an animation object if we are using keyframes
    public KeyFrameAnimation ani;

    // this function will instantiate the keyframe class based on a string
    public void addKeyFrameAnimation(string str) {
        ani = new KeyFrameAnimation(str);
    }
}
KinectJoint[] joints = new KinectJoint[25];
{% endhighlight %}

In the end you will have keyframe class instances for each joint, and you will have the number of keyframes and the positions in each keyframe for all those instances. That is the data that is interpolated to create the animation. The process would be, the animation is currently at time x, which takes place between keyframes i and i+1, so for each of the 25 joints we will interpolation 'position at keyframe i' and 'position at keyframe i+1' with 'time x'. The functions below show how the interpolation happens based on points at certain frames, these functions are called in the `Update()` function on the script that creates and updates the virtual model.

{% highlight csharp %}
// This class will be used to add animation to an object, based on keyframing
public class KeyFrameAnimation {

    // time[i] = time of keyframe i
    private float[] time;

    // position[i] = position of object on keyframe i
    private Vector3[] position;

    // total number of keyframes
    private int numberOfFrames;

    ...

    // Return a linear interpolation of the positions of frames i and j, at "time" u (0 <= u <= 1)
    public Vector3 interpolationLinearPos(int i, int j, float u)
    {
        return Vector3.Lerp(position[i], position[j], u);
    }

    // Returns a linear interpolation of the positions using the TCB method, the interpolation
    // result is between points p1 and p2, using values t, c and b, and at "time" u (0 <= u <= 1)
    // OBS: Catmull Rom interpolation = T = C = B = 0
    public Vector3 interpolateTCBPos(int p0, int p1, int p2, int p3, float t, float c, float b, float u)
    {
        Vector3 DSiplus1 = ((1 - t) * (1 - c) * (1 + b) / 2 * (position[p2] - position[p1])) + ((1 - t) * (1 + c) * (1 - b) / 2 * (position[p3] - position[p2]));
        Vector3 DDi = ((1 - t) * (1 + c) * (1 + b) / 2 * (position[p1] - position[p0])) + ((1 - t) * (1 - c) * (1 - b) / 2 * (position[p2] - position[p1]));

        return Mathf.Pow(u, 3) * (2 * position[p1] - 2 * position[p2] + DDi + DSiplus1) + Mathf.Pow(u, 2) * (-3 * position[p1] + 3 * position[p2] - 2 * DDi - DSiplus1) + u * DDi + position[p1];
    }

    ...

} // end KeyFrameAnimation class
{% endhighlight %}

Now, as mentioned before, the motion capture data can be stored in a file (which is what is used to create an animation), or it can be transmited in real time for the AR or the VR application. In the case we are using the real time data, there is no need to interpolate the positions, because the data is just rendered as soon as it is received. In the case of saving the motion capture data, the data is saved in a file with a specific format which is processed and used to create a `KeyFrameAnimation` instance.

# Displaying the Character in AR

For the AR application, the Vuforia SDK was used. It’s a SDK that includes a number of features such as recognizing and tracking targets, an object scanner, support for mobile devices and digital eyewear, and also an Unity extension.

For AR, target recognition and tracking is a key aspect, and Vuforia offers the option of doing it reliably through Image Targets. This process involves using an image as a target. When the application is running in a mobile device it will use the device’s camera to obtain a video feed, and it will look for the image target in the feed through image recognition algorithms. Once the target has been found, it will start to track its position on the video feed.

The animation of the virtual character is placed with respect to this target. The target essentially becomes the “origin” of a fixed virtual coordinate system on the real world. The virtual character is placed on the real world using this coordinate system as a base, which means that by moving the device and looking at this target in different positions and angles, the animation of the virtual character will change accordingly to still respect the coordinate system. This is what gives us the illusion of the character being a physical object on the world.

<div style="text-align:center;padding-bottom:10px;">
<iframe width="560" height="315" src="https://www.youtube.com/embed/Hmh5L7BBmJM" frameborder="0" allowfullscreen></iframe>
<div style="text-align:center;margin-top:-5px;"><font color="gray" size="2px">Running the application on a Nexus 6P, displaying a recorded animation.</font></div>
</div>

The video above shows the application in action. It runs on any Android smartphone, with it you are able to change the virtual character model and move around to see the animation play in different angles. You can pause it at any time to just see how the pose looks as well.

# Displaying the Character in VR

Another application that I also ended up developing was a way to display this character in a virtual reality experience for that I used the AR/VR glasses Epson Moverio BT-200. These glasses use Android as their OS, so it's possible to develop an Unity app for android and deploy it to the glasses.

Of course this application is different from the previous one made for AR. Here I used Moverio's plugin for Unity, which includes APIs to access sensors and other components of the headset.

In the scene I created there is a virtual stage and a starting position for the virtual character with respect to that stage. This is where the animation will be displayed, and the camera that will see this scene is replaced by the Moverio "camera". The camera is the object that is actually affected by the glasses, a script is attached to the object and it controls the orientation of the camera based on the gyroscope data from the glasses. This allows the user to see the scene in different angles depending on where he is physically looking.

<div style="text-align:center;padding-bottom:10px;">
<img src="http://i.imgur.com/MhXh0hB.png" style="float: left; width: 49%; margin-right: 0.5%">
<img src="http://i.imgur.com/PTT669v.png" style="float: left; width: 49%">
<div style="text-align:center;"><font color="gray" size="2px">Example showing how the orientation of the glasses affected the scene the user is viewing. By physically looking up the scene is updated to show you a different angle.</font></div>
</div>

# Results

In the end the whole system worked really well, without any major issues. In fact, the only two actual issues I'd say I had were technical. One was how noisy the motion capture data can be, which is visible in the animation when you see some limb going crazy in a way no human could possibly move. To fix that you'd need another a better motion capture device. Another is how the animation can "drop" frames depending on the virtual model you use, for instance, the models that use particles will affect the performance of the animation depending on the device they are running on, which means you'll need a more powerful device for more complex models.

# References

Here are some good links you can used if you are interested in any of that stuff.

* [Unity][unity] - Unity's website.
* [Microsoft Kinect][kinect] - Microsoft Kinect's website.
* [Vuforia Developer Portal][vuforia] - Vuforia's website.
* [Epson Moverio][moverio] - Moverio's website.
* [The Principles of Animation][animation] - Great article about "The Principles of Animation" by Ralph A. De Stefano.
* [Pixar in a Box][animation2] - Khan Academy and Pixar's collaboration course to teach a number of computer graphics concepts.

[unity]: https://unity3d.com/
[kinect]: https://developer.microsoft.com/en-us/windows/kinect
[vuforia]: https://developer.vuforia.com
[moverio]: https://moverio.epson.com
[animation]: https://www.evl.uic.edu/ralph/508S99/
[animation2]: https://www.khanacademy.org/partner-content/pixar
