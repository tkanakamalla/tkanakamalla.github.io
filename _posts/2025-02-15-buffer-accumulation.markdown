---
layout: post
title:  "Buffer accumulation caused accumulation of updates"
date:   2025-02-15
categories: weekly-log
---

Finally the streak, though short lived, is broken! As feared in the [beginning]({% post_url 2025-01-04-beginning %}), I missed posting in the last couple weeks partly because there was not much progress to update and I was a bit ~~lazy~~ busy in the earlier week debugging an interesting memory accumulation problem last week. Anyway, I don't want to beat myself for not being consistent, rather just be kind and tell myself it's better late than never. So, here is the (ac)cumulated update.

Let us start with what the memory accumulation problem was all about. This was from an AES67 based application, on which I [talked](https://gstconf.ubicast.tv/videos/real-time-network-audio-with-gstreamer-on-windows/) about in the [GstConf2024](https://gstreamer.freedesktop.org/conference/2024/). The Tx side of this application runs a GStreamer pipeline in which multiple `appsrc`s consume the buffers, which are further aggregated into a single buffer and wrapped with the RTP header before getting pushed downstream to a `udpsink` for AES67 multicast.

In a scenario where the pipeline clock is changed by the application dynamically, the memory usage of the process kept growing rapidly. That's it, that is the problem and it is clear that memory is not being released promptly post the clock switch.

But it took a while to reproduce the behavior on my setup and a little longer to figure out the root cause. After a day or more of iterations with logging of different categories at different debug levels and pipeline dumps at different stages, with the help of my colleague [Arun](https://arunraghavan.net), we found the reason for memory rise is the accumulation of the buffers in the `appsrc`'s queue.

At the time of a clock switch, the buffers that are already queued up to be rendered contain the timestamps as per the old clock. But after the switch, if the clock shifts backwards or to a different range of time, the sink keeps on waiting for the timestamp of the buffer to arrive, which could be in far future relative to the new clock. So the pipeline's streaming thread gets blocked until then(or forever).

![buffer_accumulation](/assets/buffer_accumulation.svg)

Unaware of this, the application keeps pushing new data to the `appsrc` and its queue keeps growing. Hence the rise in the overall process memory. While I digged a bit to figure out the appropriate resolution for this problem, surprisingly, I could not find any documentation or an example to handle similar situation in the GStreamer code or examples. So as a simple fix for this issue, I have set the pipeline to READY state instead of the PAUSED before switching the clock so that the initializations and negotiations happen freshly with the new clock. That fixes the memory rise problem but there might be small discomfort listening to the audio  during the clock switch because of the pipeline re-initialization which needs to looked at separately with fresh perspective.

Apart from this, in the PipeWire world, the MR to [avoid use of bufferpool for audio by default](https://gitlab.freedesktop.org/pipewire/pipewire/-/merge_requests/2259) has been tested and merged. Currently I am looking at fixing some issues seen while doing the video render and capture using the `pipewiresink` (provide mode) and `pipewiresrc` respectively. One of the issues is we see with raw planar video formats  like NV12:

- an artifact while rendering raw planar video formats like NV12, like below

    *the original frame*
    ![original](/assets/actual_image.png)

    *frame with artifacts as captured by the pipewiresrc*
    ![artifact](/assets/pwsink_artifact.png)

- also the video keeps freezing during the playback because the buffers are being dropped
    ```
    Redistribute latency...
    0:00:00.116156731 373320 0x7f7b64000b90 ERROR              videometa gstvideometa.c:357:default_map: plane 1, no memory at offset 2073600
    0:00:00.116170354 373320 0x7f7b64000b90 ERROR                default video-frame.c:168:gst_video_frame_map_id: failed to map video frame plane 1
    0:00:00.116176304 373320 0x7f7b64000b90 WARN             xvimagesink xvimagesink.c:1183:gst_xv_image_sink_show_frame:<xvimagesink0> could not map image
    0:00:03.337735182 373320 0x7f7b64000b90 WARN                basesink gstbasesink.c:3147:gst_base_sink_is_too_late:<xvimagesink0> warning: A lot of buffers are being dropped.
    0:00:03.337754842 373320 0x7f7b64000b90 WARN                basesink gstbasesink.c:3147:gst_base_sink_is_too_late:<xvimagesink0> warning: There may be a timestamping problem, or this computer is too slow.
    WARNING: from element /GstPipeline:pipeline0/GstXvImageSink:xvimagesink0: A lot of buffers are being dropped.
    Additional debug info:

    ```

The root cause for both the above issues seems to be the same. The video plane data, especially in case of multiple planes, is not being exchanged correctly between the `GstBuffer` and `pw_buffer` so there seems to be a problem in extracting the buffers on the receiver side i.e., `pipewiresrc`.

I wanted to get a resolution for this by this week but apparently I will be working on this issue in the early next week as well before moving on to the other issues.