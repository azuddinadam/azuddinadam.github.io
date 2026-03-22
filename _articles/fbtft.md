---
id: 0
title: "How the Linux Framebuffer works"
subtitle: "Novice's guide into the logic of fbtft driver subsystem."
date: "2026.03.18"
tags: "framebuffer, display driver"
---
Understanding how Linux displays images on small screens can be difficult
because there are so many layers of code. This article explains how the legacy
fbtft driver handles this process by acting as a bridge between the operating system
and display hardware.

We will look at how the kernel uses memory tricks to detect changes
and how it sends that data to a screen. By the end, you will understand the basic
path that data takes from a computer's memory to the physical pixels on a display.

## Prerequisites

This article assumes you have basics of programming and C,
like functions and structs. Reader should also know how a driver works,
or you can read [this article](<https://medium.com/@ahmed.ally2/understanding-linux-kernel-drivers-the-bridge-between-hardware-and-software-f3b2c1e37d90>) by Ahmed Ally for an introduction to device driver.

Before we begin, let's define a few terms. A driver is the code
that helps the kernel control specific hardware. As such, a display driver
is a driver that controls display hardware, like a monitor.

A framebuffer device (fbdev) is a subsystem in linux kernel
that handles display driver. A frame is just an array of pixels displayed
on the screen at a point in time. A buffer is a chunk of memory
that is used for something, in this case each address maps to coordinates (x,y)
on the screen. In short, a framebuffer uses certain part of memory
to map pixel data from system RAM onto the screen.

In linux, everything is a file. Other than normal file
in the file system, a node is special file that the kernel will  use to
trigger access for our driver code. For example, when the system boot,
the kernel will create a node as a file in /dev/fb0. When a user app wants to display
something to the screen, the kernel will copy data from the user space
into the kernel space where the driver lives. Then  the driver will take the data
and process it until its eventually physically rendered by the display.

Driver doesn't talk directly to the display. Instead, it talks to a component
called controller which physically lights up pixels on the display.
This controller has its own CPU and memory (Graphic RAM/GRAM),
independent from the system CPU and RAM managed by the operating system.

Each display driver in the linux kernel handles the specific quirk of
specific controller, for example translating kernel commands
into communication protocol that the controller understands like SPI or I2C.

## Okay, but how exactly does it work?

Let's look at a few structs that the kernel provided an API
for drivers managing the tiny TFT display commonly used by Arduino:

```

struct fbtft_par {
 struct spi_device *spi;
 struct fb_info*info;
 struct fbtft_ops fbtftops;
};

```

The fbtft_par consists of various parameters used for managing
the driver like references to an spi device, fb_info, and fbtft_ops.

The spi_device struct is how the kernel represent our physical connection.
It defines some rules for our communication with the controller,
such as the clock speed (how fast to send data), which SPI bus
our display is plugged into, and which Chip Select (CS) pin is used
to signal to the display that the system CPU is talking to it directly.

```
struct fb_info {
    struct fb_var_screeninfo var;  /* Variable screen info (Resolution, etc)*/ 
    struct fb_fix_screeninfo fix;  /* Fixed screen info (Buffer address, etc)*/ 
    void *screen_base;             /* The pointer to the pixels in RAM */
    struct fb_ops *fbops;          /* Generic kernel hooks */
    void *par;                     /* Pointer to our custom fbtft_par */
};
```

fb_info has lots of fixed and variable metadata from the
fb_fix_screeninfo and fb_var_screeninfo respectively.

```

struct fbtft_ops {
    int (*init_display)(struct fbtft_par*par);

    int (*write)(struct fbtft_par*par, void *buf, size_t len);
    int (*read)(struct fbtft_par *par, void*buf, size_t len);
    int (*write_vmem)(struct fbtft_par*par, size_t offset, size_t len);
    void (*write_register)(struct fbtft_par*par, int len, ...);

    void (*set_addr_win)(struct fbtft_par*par, int xs, int ys, int xe, int ye);
    void (*update_display)(struct fbtft_par*par, `unsigned int start_line, 
              unsigned int end_line);
};

```

The fbtft_ops struct contains all the operations that the fbtft driver may override
to implement logics for a specific controller. This includes:

- read and write data to a buffer in system RAM and GRAM.
- write_vmem write data from system RAM to the controller's GRAM
- write_register values to the controller's register
- update_display push updates to the physical hardware

The functionality of a framebuffer driver can be divided into 2 phases:

### Initialization

Before drawing any pixel, the driver must first wake up the display hardware.
This starts with the CS pin, which is defined in our spi_device struct.
By pulling this pin low, the CPU signals the controller that a conversation
is starting. Without this, the controller will ignore anything from the SPI bus.

Once woken up, the driver uses the write_register operation to send specific
sequences of numbers, found from the controller's datasheet,
to the controller's internal registers. These commands configure essential
hardware settings like colour format and orientation.

### Rendering

Once the display is initialized, the following cycle will begin
to render anything on the screen:

![Render Cycle Diagram](/images/fbtft_render_cycle.png)

The kernel uses a clever trick to know which regions in the display need updating.
First, the framebuffer is marked as non writable in the page table.

When an application tries to write anywhere on the screen, the CPU's MMU triggers
a page fault because the write permission is missing. The kernel
intercepts this fault, mark the page as dirty, and add it to the list
of dirty pages associated with the instance of fb_info. The kernel briefly makes
the page writeable so the application can complete its write to the RAM.

Every certain milliseconds, the fb_deferred_io_work runs. It looks at the
list of dirty pages since it last ran and calculates the combined bounding box
(by setting the start and end coordinates; xs, ys, xe, and ye) which covers all
the dirty pages. Finally, it calls update_display to push the changes to the hardware.
Once finished, the kernel resets the pages to non-writeable to
catch the next round of faults.

Now let's go into the update_display method. It will consecutively call
two important function, set_addr_win and write_vmem.

The set_addr_win function tells GRAM exactly which region
we are going to update using the coordinates (xs,ys,xe and ye)
calculated from the deferred I/O work (so it doesn't update the whole screen!).

To do this, the driver uses the write_register operation. This sends
specific sequence of commands found in the datasheet, which tells the hardware
to stop what it's doing and prepare to receive data from these specific coordinates.

Finally, write_vmem is called to write actual data from the system RAM
to the GRAM using the also overrideable write operation. The driver may perform
specific logic at the byte level, like converting 32-bit colour to 16-bit colour,
or flipping the byte order of the data, since most modern CPUs uses Little Endian
while most SPI display controller uses Big Endian. These details are unique
to the driver and can be found from the controller's datasheet.

## Why is it "Legacy""?

While the framebuffer system is great for learning the basics, it's considered legacy
because fbdev is designed in the simpler era of computing. It has no support for
hardware acceleration, meaning the CPU do all the heavy lifting for every
pixel change. It also assumes 'flat' image, meaning it cant easily handle modern
UI features like multiple planes where a cursor plane overlay the background plane.

Modern Linux systems have moved toward the Direct Rendering Manager (DRM).
Unlike the simple framebuffer, DRM take advantage of the dedicated GPU that most
modern display has to handle complex layering, while also reducing the CPU workload.
In the next article, we will explore how to transition these concepts into
the modern DRM/KMS world.
