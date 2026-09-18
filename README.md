## Summary
When trying to directly drive more than one display (via VK_KHR_display) from a single VkInstance, either swapchain creation fails or vkQueuePresentKHR hangs indefinitely. Simultaneously using multiple VkInstances, one for each display, works around the issue.

## Machine

OS: Windows 11 Pro for Workstations, version 10.0.26200 Build 26200

Hardware:
- NVIDIA RTX A4500
- NVIDIA RTX 45000 Ada Generation
- Dell P2715Q (detached left display) 4k 60Hz
- ROG XG27UQR (detached right display): 4k 144Hz
- Monitor for desktop: 1920x1200 60Hz

Drivers tested: 
- 597.06 (latest studio)
- 597.11 (latest Vulkan beta)
- 616.92 (latest new feature)

## Setup

I followed the instructions to compile and run the vk_ddisplay sample app at commit 29ca3f0: https://github.com/nvpro-samples/vk_ddisplay. I created a config file that defined a 32:9 canvas and put display 0 on the left and display 1 on the right:
```json
{
    "canvas": { "fov": 90, "aspectNum": 32, "aspectDen": 9 },
    "displays": [
        { "index": 0, "canvasOffsetX": 0, "canvasOffsetY": 0, "canvasWidth": 0.5, "canvasHeight": 1.0 },
        { "index": 1, "canvasOffsetX": 0.5, "canvasOffsetY": 0, "canvasWidth": 0.5, "canvasHeight": 1.0 }
    ]
}
```

I then tested two different hardware configurations (splitting the monitors across two GPUs vs. putting them on the same GPU), each across the three driver versions I mentioned:

## Split Configuration
- Topology:
    - A4500 connected to Dell and desktop monitor
    - 4500 Ada connected to ROG monitor
- Behavior on first run:
    - First device initializes okay (and the left display turns on)
    - hits InitializationFailedError while initializing the second device from createSwapchainKHRUnique at vk_ddisplay/logical_display.cpp:229
- Behavior on second run:
    - Appears to initialize successfully - control window appears.
    - Left display turns off (was previously on but black from the first run)
    - Right display shows a frame then freezes.
    - Control window is also completely unresponsive.
    - The application cannot be shut down at this point, and the system must be rebooted to continue.
- Identical results were observed across all drivers.

## Shared Configuration
- Topology:
    - A4500 connected to desktop monitor
    - 4500 Ada connected to Dell and ROG monitors
- Behavior on first run:
    - First display initializes okay (and the left display turns on)
    - hits InitializationFailedError while initializing the second display from createSwapchainKHRUnique at vk_ddisplay/logical_display.cpp:229
- Behavior on second run:
    - Appears to initialize successfully - control window appears.
    - Left display turns off, right display remains off.
    - Control window is completely unresponsive.
    - The application cannot be shut down at this point, and the system must be rebooted to continue.
- Identical results were observed across all drivers.

## Additional Testing
- I also tried running two instances of vk_ddisplay.exe simultaneously, each with a config file to drive one of the two monitors. In all combinations of hardware configuration and driver, this worked fine.
- In the application I was developing when I initially encountered this issue, I found that using two VkInstances in the same application (one for each display/device) also works fine. It is only if both displays/devices are created from the same instance that this problem occurs.
- Also in the application I was developing, I found that in the cases where the application hangs, one render thread is permanently stuck waiting for vkQueuePresentKHR to return.
