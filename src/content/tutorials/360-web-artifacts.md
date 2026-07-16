---
title: "Build a 360 view or Exploded view Scroll Animation"
description: "Create a scrolling effect animation for any object."
category: "Web Artifacts"
pubDate: 2026-07-16
difficulty: "beginner"
---

**Step 1: Sign up for Google Flow for a free tier account.** 

Go to [Google Flow](https://labs.google/fx/tools/flow)

<br>

**Step 2: Generate your first and last frames in Google Flow.**

**Note**: Replace **[PRODUCT]** with your own object. 

Start frame (your product, assembled):

```text
High-end studio product photograph of **[PRODUCT]**, fully assembled and intact, floating dead center against a seamless matte light-grey studio background. Straight-on view, perfectly level camera, soft even studio lighting, one subtle soft shadow directly beneath the product, photorealistic, ultra sharp, high detail. Keep the product small enough in frame that there is generous empty space on every side. No text, no watermark, no hands, no props, no brand logos. Landscape 16:9.
```

End frame (your product, taken apart):

```text
High-end studio product photograph of the same **[PRODUCT]** as a clean exploded view: the outer shell or casing lifted apart and every major part and layer separated and floating in an organized, evenly spaced arrangement along one axis, like a technical teardown render. Same straight-on view, same perfectly level camera, same soft even studio lighting, same seamless matte light-grey studio background, one subtle soft shadow beneath. The core body of the product stays in the exact center at the same size, with the parts spreading outward around it. Photorealistic, ultra sharp, high detail. No text, no watermark, no hands, no props, no brand logos. Landscape 16:9.
```

<br>

**Step 3: Animate the in-between with Frames to Video in Google Flow.**

Choose Frames to Video: First + Last, upload both images, and add a short motion prompt (for example, "slow smooth 360 rotation") and download the clip.

```text
The product slowly comes apart into an exploded view: the casing separates and the internal parts spread outward smoothly and evenly. Camera locked in place, background unchanged.
```

<br>

**Step 4: Slice the clip into frames in EZGIF**

- Go to [EZGIF](https://ezgif.com/video-to-jpg)
- Upload your video
- Select **Video to JPG (image sequence)** from dropdown menu
- Select Frame rate (FPS): 30
- Check the box for "Don't display the images, I just want a zip file"
- Click the **Convert to JPG!** button
- Unzip the folder containing the numbered sequences of images


<br>

**Step 5: Give your AI agent access to the folder containing your image sequence.**

Web artifact prompt:

```text
Using this folder of image frames, build a scroll-linked image sequence on an HTML canvas. Preload all the frames first, then map the scroll position to the frame index so the animation plays frame by frame as the user scrolls, the same technique Apple uses on the AirPods page. Use GSAP and ScrollTrigger. Pin the canvas while the sequence plays, scrub it smoothly to scroll so there's no stutter, keep it sharp and centered on mobile, and add a graceful fallback if the frames are still loading. Give me a single self-contained file I can drop straight into my site.
```

---

## 360 Scroll Example

I wanted to create a network/server rack using stock images. I provided the front and back images for each device then used the following prompt to generate a front and back view of the rack. 


**Step 1: Create the first and last frame in Google Flow.**

Go to [Google Flow](https://labs.google/fx/tools/flow)

First frame prompt:

```text
Create a front view of a completed network/server rack. The layout of the rack and order of devices is below. Attached is visual mockup of the layout is called rack-design.png, and front and back images of each device for your reference.

Rack Configuration:
x1 fiber-patch-panel.jpg
x1 copper-patch-panel.png
x2 cisco-asa-firewall.jpg
x2 Cisco-Catalyst-3850-L3-Switch.png
x2 ribbon-scb-5400.png
x1 Cisco-897-Router.png
x1 MOXA-Console-Server.png
x1 Dell-Poweredge-R620-voip-monitor.png
X1 Dell-VRTX-Blade-Server.png
x2 APC-Switched-Rack-PDU.jpg
```

<img alt="reference images" src="/360-tutorial/step1a.png" width="50%" />


Once you have the front view simply ask it to render a back view.

<img alt="create images" src="/360-tutorial/step1b.png" width="50%" />

<br>

**Step 2: Animate the in-between with Frames to Video in Google Flow.**

Drop the front and back images into the editor in order.


<img alt="select video" src="/360-tutorial/step2a.png" width="50%" />

Give the AI a motion prompt:

```text
Slow smooth 360 rotation. End on the first frame. Camera locked in place, background unchanged.
```

Download your video. 

<img alt="360 video" src="/360-tutorial/step2b.png" width="50%" />


<br>

**Step 3: Go to EZGIF to slice up your video.**

- Go to [EZGIF](https://ezgif.com/video-to-jpg)
- Upload your video.
- Choose the settings listed above and download the images. 


<img alt="ezgif" src="/360-tutorial/step3.png" width="50%" />

<br>

**Step 4: Give your AI agent access to the folder containing your image sequence.**

Web artifact prompt:

```text
Using this folder of image frames, build a scroll-linked image sequence on an HTML canvas. Preload all the frames first, then map the scroll position to the frame index so the animation plays frame by frame as the user scrolls, the same technique Apple uses on the AirPods page. Use GSAP and ScrollTrigger. Pin the canvas while the sequence plays, scrub it smoothly to scroll so there's no stutter, keep it sharp and centered on mobile, and add a graceful fallback if the frames are still loading. Give me a single self-contained file I can drop straight into my site.
```

<br>

**Click on the image below and scroll up using your mouse to see results.**

<div class="sequence-wrapper">
  <iframe 
    src="/360-tutorial/360.html" 
    style="width: 50%; height: 80vh; min-height: 480px; border: none; border-radius: 16px; box-shadow: 0 10px 30px rgba(0,0,0,0.15);"
    allowfullscreen>
  </iframe>
</div>
