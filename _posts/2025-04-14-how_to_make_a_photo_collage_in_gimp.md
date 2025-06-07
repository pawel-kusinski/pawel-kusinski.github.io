---
title: "How to Make a Photo Collage in GIMP"
categories:
  - Misc
tags:
  - GIMP
---

## What We Will Make

In this post, I will explain how to make a simple photo collage in GIMP.

**Project details:**
- We have 4 images. Each image is in landscape orientation and has a 4:3 aspect ratio.
- We want to create a 2 × 2 horizontal collage, also with a 4:3 aspect ratio.
- The collage resolution is 1024×768.
- There are no borders or spaces between the images.

## Steps

### Step 1: Create the Canvas and Define the Size

1. Click on **File -> New...**
2. A window called **Create a New Image** will appear.
3. Enter the dimensions in the **Image Size** section: *1024* for **Width** and *768* for **Height**.
4. Click the **OK** button.
![gimp_collage_create_a_new_image.png](/assets/images/gimp_collage_create_a_new_image.png)

### Step 2: Create Guidelines

1. Click on **Image -> Guides -> New Guide (by Percent)...**
2. A window called **Script-Fu: New Guide (by Percent)** will appear.
3. Set **Direction** to **Horizontal** and **Position (in %)** to *50.00*. These are the default values, so just click the **OK** button.
4. A horizontal dashed line dividing the canvas in half will appear.
5. Click on **Image -> Guides -> New Guide (by Percent)...** again.
6. The **Script-Fu: New Guide (by Percent)** window will appear again.
7. Set **Direction** to **Vertical** and keep **Position (in %)** at *50.00*. Click the **OK** button.
8. A vertical dashed line dividing the canvas in half will appear.
9. Make sure **View -> Snap to Guides** is enabled.
![gimp_collage_guidelines.png](/assets/images/gimp_collage_guidelines.png)

### Step 3: Open Images

1. Click on **File -> Open as Layers...**
2. A window called **Open Image as Layers** will appear. Select your images and click the **Open** button.
3. Each image will appear as a layer in the **Layers** panel.
![gimp_collage_layers.png](/assets/images/gimp_collage_layers.png)

### Step 4: Arrange the Images

1. Click on the layer containing an image. Press **Shift+S** to scale.
2. A **Scale** window will appear.
3. Enter *512* in the **Width** field and press **Enter**. The **Height** will adjust automatically to maintain the aspect ratio. Press the **Scale** button to confirm.
4. Repeat steps 1–3 for the remaining three images.
5. Use the **Move Tool** to arrange the images so they align with the guidelines.
6. If an image is outside the canvas boundaries, it can still be selected using the **Move Tool**. It will appear as a yellow outline, as shown in the screenshot below.

<figure class="half">
	<a href="/assets/images/gimp_collage_scale.png"><img src="/assets/images/gimp_collage_scale.png"></a>
	<a href="/assets/images/gimp_collage_arranging.png"><img src="/assets/images/gimp_collage_arranging.png"></a>
</figure>

### Step 5: Export the Collage

1. Click on **File -> Export As...**
2. An **Export Image** window will appear.
3. Expand **Select File Type (By Extension)** and choose the desired format (e.g., *PNG*, *JPEG*).
4. Enter a file name, choose a location, and click the **Export** button.
5. Depending on the format, an additional window may appear with format-specific settings.
6. Click the **Export** button again. You're done!
![gimp_collage_export.png)](/assets/images/gimp_collage_export.png)

## Notes

These guidelines were written and tested using GIMP 2.10.34.
