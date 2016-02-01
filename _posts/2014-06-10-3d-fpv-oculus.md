---
layout: post
title: "Stereo Vision with Oculus Rift"
description: "Making a stereo vision camera and real-time IP video with Oculus Rift"
tags: [three.js, 3d, drones, oculus]
image:
  feature: 3dfpv.jpg
---

I was fortunate to work on this project as part of a hackathon at work.
I've always had an interest in FPV flying, and I have a pair of FatShark goggles
that I've use to fly with a mini CMOS NTSC camera.
The quality is poor, and it can be difficult to gauge your surroundings as there are no
depth cues. It works, but is far from ideal or immersive.

Some people online have tried to replicate a real-time view of their surroundings using stereo cameras
and the Oculus Rift. I had a spare devkit I ordered awhile ago of the DK1, so I decided to give it a shot,
trying to go for a pure javascript approach. I only had 3 days, so I wasn't able to arrive at a perfect solution
but it did work.

##The Hack

I needed this hack to be on the cheap, and since stereo cameras are expensive I had to find a way to make
my own, using two cheap chinese IP cameras I had laying around. The idea is to run the cameras via the datalink on the drone
to the ground, and use ffmpeg to serve the video over a web socket to the browser for each eye, where I can apply
barrel distortion using webgl.

Or so that's how it's supposed to go. Ignoring timing issues with the camera left and right frames and no way to reliably sync them,
it actually almost had hope of working and produced some initially good results testing indoors.

Following the 1/30th rule of stereography, which states that the interaxial separation (distance between eyes, or in this case camera lenses)
should be no more than 1/30th the distance of the closet subject, or in this case about 2.5 inches to have features of about 75 inches away and background
at infinity. Convergence of the two final images can then be adjusted while wearing the rift, as well as doing the necessary barrel distortion required
due to the optics in the rift (what give the immersive feeling).

<figure class="half">
	<img src="/images/cameras.jpg" title="rtsp ip cameras">
	<img src="/images/mount.png" title="3d printed mount">
	<figcaption>Cheap cameras and their cheap mount</figcaption>
</figure>

##RTSP In The Browser

After hearing about a cool pure javascript mpeg decoder called jsmpeg, I wanted to try to do all the video processing in the browser. This meant somehow getting the video to the browser from the RTSP stream offered by the cameras. Unfortunately, although RTSP is a standard transport for video streaming and supports several standard video formats, it’s not possible to natively use in the browser without some kind of plugin, typically flash.

No worries, though. FFMpeg, the swiss-army knife of streaming video and video transcoding comes to the rescue with the ability to transform a conventional mpeg1 streaming video transport that we can use directly with jsmpeg. In fact, someone had already created the node module [node rtsp stream](https://www.npmjs.com/package/node-rtsp-stream) that wraps and handles the ffmpeg process creation and sends the data over a websocket!

In my case, I had two cameras so I simply had two servers and two different node/ffmpeg processes, each running on their own websocket:

{% highlight js %}
var Stream = require('node-rtsp-stream');

// left eye
var stream = new Stream({
  name: 'video1',
  streamUrl: 'rtsp://192.168.1.11:554/user=admin&password=&channel=1&stream=0.sdp?real_stream--rtp-caching=100',
  wsPort: 9991
});

// right eye
var stream = new Stream({
  name: 'video2',
  streamUrl: 'rtsp://192.168.1.11:554/user=admin&password=&channel=1&stream=0.sdp?real_stream--rtp-caching=100',
  wsPort: 9992
});
{% endhighlight %}

Decoding MPEG1 video in the browser actually works quite well thanks to jsmpeg, which takes data in via websockets and renders each frame to a canvas element. Using three.js, it is
possible to create a texture from the contents of the canvas with video data:

{% highlight js %}
// create client and stream to canvas element
var client = new WebSocket('ws://127.0.0.1:9991'),
    canvas  = document.createElement('canvas'),
    player = new jsmpeg(leftClient, { canvas: canvas });
{% endhighlight %}

{% highlight js %}
// create texture from canvas content
var texture = new THREE.Texture(canvas);
texture.magFilter = THREE.LinearFilter;
texture.minFilter = THREE.LinearFilter;
texture.format = THREE.RGBFormat;
{% endhighlight %}

##Ready for VR: Lens Distortion

<figure class="half">
	<img src="/images/barrel.png" title="barrel distortion">
	<img src="/images/pin.png" title="pincushion distortion">
	<figcaption>Rift Lens Distortion</figcaption>
</figure>

You may have noticed using a headset like the rift there is a lot of distortion when viewing content that isn't specifically designed for it, and vice versa. You're probably familair with the oculus screenshots showing two rounded views of images. That's because the lenses apply a pincushion distortion, which is an optical trick to make it seem like a flat image is actually immersive. In order to reverse the effect, a barrel distortion must be applied to the image.

This is easiest done with a fragment shader. A shader is best described as a simple program that operates on either all verticies of the 3D scene or all rendered pixel values. These programs are run in parallel on the GPU for each frame of the scene (and pixel/vertex) so they are quite handy and well suited for the task. In my case, I was interested in the fragment shader.

The fragment shader operates on pixel values, so applying perspective transformations and my case barrel distortion can work in real-time, as fast as the video stream. Here's a fragment shader to do a simple barrel distortion and the accompyning vertex shader:

Vertex shader:
{% highlight html %}
<script id="vertexShader" type="x-shader/x-vertex">
  varying vec2 v_pos;

  void main() {
    v_pos = vec2(position.xy);
    gl_Position = vec4(position, 1.0);
    gl_PointSize = 1.0;
  }
</script>
{% endhighlight %}

Fragment shader:
{% highlight html %}
<script id="fragmentShader" type="x-shader/x-fragment">
  precision highp float;
  uniform float barrel_power;

  vec2 distort(vec2 p) {
    float theta  = atan(p.y, p.x);
    float radius = length(p);
    radius = pow(radius, barrel_power);
    p.x = radius * cos(theta);
    p.y = radius * sin(theta);

    return 0.5 * (p + 1.0);
  }

  varying vec2 v_pos;
  uniform sampler2D video;

  void main() {
    vec2 d = distort(v_pos);
    if (d.x > 1.0 || d.x < 0.0 || d.y > 1.0 || d.y < 0.0) {
      gl_FragColor = vec4(0.0, 0.0, 0.0, 1.0);
    } else {
      gl_FragColor = texture2D(video, d);
  }
}
</script>
{% endhighlight %}

The video in the first step of the jsmpeg client is being rendered to the canvas, element but we need a way to get it into a webgl texture so the fragment shader can do its work.
To do this, simply read the canvas raw RGB image data and create a texture in three.js:

{% highlight js %}
// create texture from canvas content
var texture = new THREE.Texture(canvas);
texture.magFilter = THREE.LinearFilter;
texture.minFilter = THREE.LinearFilter;
texture.format = THREE.RGBFormat;
{% endhighlight %}

Video data is now being sent to the GPU in the form of a texture. Now, create a new three.js material with the barrel distortion shaders:

{% highlight js %}
var vertShader = document.getElementById('vertexShader').innerHTML;
var fragShader = document.getElementById('fragmentShader').innerHTML;

var left = new THREE.ShaderMaterial({
  uniforms: {
    video: {
      type: 't',
      value: texture
    },
    barrel_power: {
      type: 'f',
      value: 1.5
    }
  },
  vertexShader: vertexShader,
  fragmentShader: fragmentShader,
  wireframe: false
});
{% endhighlight %}

Now, simply create a flat plane geometry of arbitrary size (we're using shaders to compose our image so it doesn't really matter) and create a mesh of the geometry and material:

{% highlight js %}
var plane  = new THREE.PlaneGeometry(256, 256, 4, 4);
var mesh = new THREE.Mesh(plane, material);
{% endhighlight %}

In standard three.js parlance, let's create a camera, scene, renderer and insert the three.js webgl canvas to the document:

{% highlight js %}
var camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 1, 10000);

var scene = new THREE.Scene();
var renderer = new THREE.WebGLRenderer();

renderer.setPixelRatio(window.devicePixelRatio);
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);
{% endhighlight %}

Pulling it all together, the last thing to do is to setup the render loop. On each render, we want to update the texture and render the webgl frame:

{% highlight js %}
function render() {
  texture.needsUpdate = true;
  renderer.render(scene, camera);
  requestAnimationFrame(render);
}
{% endhighlight %}

How well does it work? If you have a modern browser you can check it out right here! Here's an example of streaming a static video file instead of live camera (aspect ratio and chromatic abberation are not corrected in this example):

<iframe height="360" width="640" src="/demos/barrel_distortion.html"></iframe>

## Left & Right

Now that we've demonstrated the ability to apply a simple shader to our video, we can use a more complete example that takes care of things like chromatic abberation caused by the lens and handles both cameras. This is accomplished by
drawing both video feeds to an enlarged canvas, side by side:

{% highlight js %}
function render() {
  texture.needsUpdate = true;
  //left
  ctx.drawImage(video, 0, 0, size.width, size.height);
  //right
  ctx.drawImage(video, size.width, 0, size.width, size.height);
  renderer.render(scene, camera);
  requestAnimationFrame(render);
}
{% endhighlight %}

I've also used a [more complete rift shader](https://github.com/evanw/oculus-rift-webgl/blob/master/distortion.html) that handles chromatic abberation and the center spacing in the rift optics and LCD screens, but the same basic concepts hold true.

<iframe height="360" width="640" src="/demos/oculus.html"></iframe>

## Test Flight

I mounted my camera rig on a gimbal and was able to do a few test flights before going for a fly at Treasure Island. The link drops out in some places, but overall worked well. The effect was certainly immersive and has given me the motivation to continue to work on the project. Here's a rather compressed video of the flight:

<iframe src="https://player.vimeo.com/video/153835562" width="500" height="304" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen></iframe>
<p><a href="https://vimeo.com/153835562">fpv</a> from <a href="https://vimeo.com/jleppert">Johnathan Leppert</a> on <a href="https://vimeo.com">Vimeo</a>.</p>

I have plans to use a board like [Nvidia's Jetson](http://elinux.org/Jetson_TK1) and run the necessary code directly on the GPU, on the drone itself and send back a composite stream suitable for viewing directly in a headset, possibly using USB 3.0 high speed
cameras, or using the built-in MIPI camera interface, which allows direct frame access on the GPU.

I posted the code of my hack to [Github](https://github.com/jleppert/3dfpv).
