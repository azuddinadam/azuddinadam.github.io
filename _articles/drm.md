---
id: 1
title: "How Linux DRM works"
subtitle: "Similarities and differences with the legacy fbdev"
date: "2026.03.21"
tags: "DRM, Driver, KMS"
---

Modern hardware has upgraded capabilities, so driver code need to utilize this.
If you don't know what a framebuffer device is, read this prev article first.



## KMS 
- set metadata like resolution and refresh rate 

## GEM 
- allocate usable buffer and return an identifier

## Panels


## CRTC 
- merge panels into pixel data 

## Encoder 
- translate pixel data to actual signal to physically light up the pixels.

