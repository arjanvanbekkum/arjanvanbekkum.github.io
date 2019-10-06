---
layout: post
title: "Augmented Reality with javascript"
date: 2019-10-06
summary: "This blog is about how to use AR.js in a website and showing some cool features of Augmented Reality."
---

Augmented Reality is becoming the next best thing when creating apps and websites. More and more companies are using it to create the next level of experience. How hard is it to get started on AR, what do you need and how does it work in general. All questions we had before we started, it seems there are many libraries on the internet to get you started. We decided to keep it relatively easy and create a website.

We decided to use AR.js as a javascript library, and it is relatively simple to use. In this blog I'm using three different AR types. We are going to create a 3D model without any animation, play a video and last, but not least we use an animated 3D model. So let's get started. 

## Library
As mentioned before, we use the AR.js library. So when creating our HTML page we need to add the javascript to our HTML page. So, in this case, we need a few lines and add them to the `<head>` tag. 
```html
    <script src="https://aframe.io/releases/0.9.2/aframe.min.js"></script>
    <script src="https://cdn.rawgit.com/jeromeetienne/AR.js/1.7.7/aframe/build/aframe-ar.js"></script>
    <script src="https://raw.githack.com/donmccurdy/aframe-extras/master/dist/aframe-extras.loaders.min.js"></script>
```

## Scene
First of all, you need to create a scene; this is the place where your AR model is going to be. No matter what library you are using, you will need the this scene and the upcoming markers. Using the AR.js library we need to add the scene to our website. So in the body tag add the following line of code 

```html
  <a-scene embedded arjs='sourceType: webcam; debugUIEnabled: false; detectionMode: mono_and_matrix; matrixCodeType: 3x3;'>
```

Because we want to use the camera on your phone or laptop, we provide it with the sourceType `webcam` attribute. 

## Assets
So next up, is to define the assets we need to show or play something in our scene later on. In this case we add a video to our assets. Add the next line of HTML code in the `<a-scene>` tag to add the asset to the 

```html
    <a-assets>
        <video crossorigin="anonymous" id="videoID" autoplay loop="true" type="video/mp4" preload="auto" src="url to the video">  
    </a-assets>
```

## Markers
To make the magic work, we need a marker to trigger the video or show the 3D model. A marker is something the AR library recognizes and then perform some action. The marker is a pattern containing color codes; the AR library uses the camera to scan the page and comparing it to the markers on the page. If it finds a match, it displays the model or shows the video. The AR.js library comes with a few predefined markers, but you can create your own. There are some rules in creating a marker; one of them is you are not able to use the color white. On our page we use two custom markers and one predefined marker. 

To use a custom marker, add to following line of code to the `<a-scene>` tag in the HTML page. 

```html
    <a-marker type="pattern" preset="custom" url="url to the marker">

    </a-marker>
```

So with all the plumbing in place lets now create a 3D model and show it.

## 3D model
Of course, you can create a fancy 3D model using advanced software. In this case we keep it simple and create a model using HTML5. We use a box with a knot to display on marker scan.

The code below shows the box, place this code in the `<a-marker>` tag. 

```html
    <a-box position='0 0.5 0' material='opacity: 0.5; side: double'>
		<a-torus-knot radius='0.26' radius-tubular='0.05'></a-torus-knot>
    </a-box>
```

Scan the marker using your phone or webcam, and the 3D model shows, moving the camera away from the marker hides the model again. 

You can try it here, scan the QR code to go this page on your phone, allow access to the camera and you are ready to go

<img src="../images/qr.png" alt="QR" width="200"/>

Next, scan this marker to display the knot in a box. 

<img src="../images/epicshitmarker.png" alt="epic shit marker" width="200"/>

## video
Creating a video is almost the same as creating the 3D model. To display the video, we use the asset we just added to our page. Create another marker with a different pattern. The marker needs an extra attribute to make the video play when scanning the marker. It needs to look like this.

```html
    <a-marker vidhandler preset="custom" type="pattern" url="url to the pattern">
```

Key is the `vidhandler` attribute, this point to a script in the HTML, which plays the video automatically when the marker is scanned. In the `<head>` tag in the HTML page add an `<script>` tag and add the following piece of code.

```javascript

    AFRAME.registerComponent('vidhandler', {
        // ...
        init: function () {
            this.toggle = false;
            this.vid = document.querySelector("#videoID")
            this.vid.pause();
        },
        tick: function () {
            if (this.el.object3D.visible == true) {
                if (!this.toggle) {
                    this.toggle = true;
                    this.vid.play();
                }
            } else {
                this.toggle = false;
                this.vid.pause();
            }
        }
    });
```

This code registers a component on the page, making the video play automatically. In the newly created marker, add the following line of HTML code. 

```html
    <a-video src="#videoID" width="1.4375" height="1" rotation="270 0 0"></a-video>
```
This will work exactly the same as the 3D model, scan this marker to play the video. 

<img src="../images/xmarker.png" alt="x-marker" width="200"/>

## 3D animated model
The coolest thing about AR is you can use it to combine the real world with models. It gets even more cooler when the models "come to live" and can walk around on your table. To add an animated 3D model to our scene you have to create one online. We use a gltf model in the new marker. In this case we use the `hiro` marker in the AR.js library; this is a predefined one. 

```html
    <a-marker id="memarker" preset="hiro">
        <a-gltf-model src="url to the gltf model" position="0 0 0" rotation="0 0 0" scale="0.03 0.03 0.03" animation-mixer="clip: Take 001; loop:repeat"> </a-gltf-model>
    </a-marker>
```

You can use all kinds of attributes to scale or rotate the model, but also use `loop` and `position` to customize the model. So scan this marker to display the animated 3D model and make it walk around a little.  

<img src="../images/HIRO.jpg" alt="hiro maker" width="200"/>

We found that it was no that hard to use this library and enjoyed playing around a little.  

With special thanks to my colleages at <a href ="http://www.xpirit.com">Xpirit<a> !!




