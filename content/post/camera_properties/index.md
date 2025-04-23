---
title: "Camera Properties Every Vision Engineer Should Know"
date: 2025-04-23
image: cover.webp
categories:
    - Learning
tags:
    - Computer Vision
---

When I started working with cameras as an engineer, I struggled finding a definitive educational resource for my needs. Most resources target very specific domains, such as photography and filmography, requiring an engineer to piece together important details themselves.

In this article, I'll share some key concepts and camera properties that you should know when starting out with a vision-oriented engineering project.

## Camera Properties

### Resolution, Frame Rate, and Bit Depth

The basic principle of a modern digital video camera is to capture some scene in real-time. To accomplish this, the camera must set a distinct resolution for the visual quality, a frame rate to determine how often

- **Resolution:** Determines the visual quality. Higher resolution provides detail but increases data rates and processing load. Typical resolutions with modern cameras fall in the range of 720p, 1080p, 2k, 4k.
- **Frame Rate (FPS):** The measurement of how often a camera captures a single image. Video is created by stitching together this series of images. Higher FPS reduces motion blur and improves real-time tracking but increases bandwidth and processing demand.
- **Bit Depth:** The amount of bits used to store a single pixel's color value. Higher bit depth improves color accuracy but increases data size.
  
### Rolling Shutters vs. Global Shutters

The shutter is a core concept for understanding cameras. In a classic film camera, the shutter is a physical window that opens and closes, exposing the film to incoming light.

While industrial video cameras, may not have a physical shutter, they do maintain the concept of sampling the camera's light sensors at a given interval. Light is accumulated by the camera's sensors for a given duration, then the value is reset based on exposure time.

There are two main types of shutters in industrial cameras, and each significantly impacts the outcome of the image capture:

- **Rolling Shutter**: Captures each frame line-by-line sequentially. This can result in distortion or skew ("rolling shutter effect") when recording fast-moving objects.

- **Global Shutter**: Captures all pixels simultaneously, eliminating distortion and providing accurate representation of moving objects. Preferred in high-speed robotics and precise measurement applications.

### Exposure Time / Shutter Speed

Shutter speed impacts how long the camera's sensors are exposed to light. The length of this period is called exposure time.

- **Long exposure:** Increases brightness and improves overall image quality in low-light conditions, but introduces significant motion blur.
- **Short exposure:** Produces sharp, clear images of fast-moving objects but results in darker images and potentially increased noise.

The objective when setting exposure time is to adjust visual quality with changing environments. In addition, fast-moving systems like those found in robotics often require short exposures to minimize blur and reduce latency, raising the need for better lighting or higher sensor gain.

### Gain

Gain amplifies the light signal received from the sensor. The units for this quantity are either in decibels (dB) or ISO. While increasing this value can counteract shorter exposure times, it can also add a significant amount of noise.

Many camera can automatically adjust gain based on lighting conditions, but this can add some amount of latency depending on the methods used. Gain can also be dynamically controlled by an application, which may be able to handle dynamic changes in a way that aligns closer to a system's requirements.

### Dynamic Range

Dynamic range is the ratio between the brightest and darkest elements a camera can simultaneously capture without loss of detail. Real-world scenarios, such as outdoor environments or headlights against dark backgrounds, exhibit high dynamic ranges.

Cameras with limited dynamic range can clip highlights or lose shadow details, negatively affecting visual analysis accuracy.

### Lens Properties

Camera lenses directly influence captured images. Relevant parameters include:

- **Focal Length:** Determines field of view (wide-angle for broader scenes, telephoto for distant detail).
- **Aperture (f-stop):** Controls the amount of light entering and impacts depth of field; lower numbers (wider aperture) allow more light but shallow depth of field.

**Typical Lens Usage:**
- Robotics navigation: Wide-angle lenses (e.g., 90° to 120° FOV).
- Detailed inspection tasks: Narrow field-of-view lenses with higher focal lengths.

### Sensor Types

There are two primary sensor technologies you will find in most cameras geared for engineering platforms:

- **CMOS (Complementary Metal-Oxide Semiconductor):** Lower power consumption, rapid readout speeds, common in industrial cameras.
- **CCD (Charge-Coupled Device):** Superior image quality in low noise conditions, traditionally favored for scientific applications.

## Common Distortions

### Motion Blur

Motion blur refers to the visual blurring of moving objects resulting from long exposure times relative to their motion speed. While this effect can be aesthetically pleasing for more artistic fields, it can negatively impacts systems that require accurate visual data. Motion blur can degrade features and imped both human perception and computer vision algorithms. It is best to minimize motion blur when configuring a video solution.

To reduce motion blur, exposure time can be reduced or frame rate can be increased. However, both create additional processing/bandwidth challenges.

### Noise

Noise is the distortion of an image, typically with grainy or other visual artifacts. It involves a loss of detail, typically from an inadequate camera configuration relative to it's environment.

To reduce noise, the source of the issue must be addressed. If not enough light is being received by the camera's sensors, exposure time may need to be lengthened and gain lowered.

### Over Exposure or Under Exposure

Intense highlights or an incredibly dark image typically means the exposure rate is not set properly for the current environment.

### White Balancing Issues

White balancing issues display themselves as an odd color tint across the entire video frame. While some cameras should typically be able to address these issues with internal processing, some cameras do not.

If this distortion cannot be handled by the camera itself, it will have to be handled on the application side when video frames are received. Sometimes static white balancing color shifts are sufficient, when a particular camera has a constant color shift that does not change. Other times, a dynamic white balancing algorithm may be necessary.
