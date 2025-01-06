---
layout: post
title:  "Getting better at GstPipeWire to make it better"
date:   2025-01-04
categories: weekly-log
---
In the past few weeks I have been working on Pipewire’s GStreamer src/sink plugins, trying to fix some outstanding [issues/bugs](https://gitlab.freedesktop.org/pipewire/pipewire/-/issues/?label_name%5B%5D=gstreamer).

Particularly in the last couple of weeks, I was trying to fix an issue - [pipewiresink: assertion 'gst_buffer_is_writable (buffer)' failed](https://gitlab.freedesktop.org/pipewire/pipewire/-/issues/1326).

To give a brief about this issue, it happens in a gstreamer pipeline when we perform multiple seek events (skip, fast-forward, rewind etc.). After a round of investigation, I found that the main cause of this issue is of two levels

- the buffers dequeued by playback client (`gstpipewiresink`) for filling data are not pushed back to the `stream`'s queue if they are not `render`ed by the `gstbasesink` during a flush/seek event, instead they are dropped/released. So after a point of the time, and multiple flushes, the queue becomes empty so the client(`gstpipewiresink`) can no longer acquire new buffers and so it cannot produce and push new data to the `stream` for the playback
- since no there are no new buffers being pushed to the `stream`, the data loop remains blocked and will not be invoking the `process` method, so the `acquire_buffer` is blocked forever

As a first step to address this problem, I had to avoid the starvation of the `acquire_buffer`, so the queue should not be empty. For this, I have added a new stream API to `return` the released/dropped buffers back to the stream’s queue so that they can be available for the next pop/dequeue calls.

With this, the queue no longer gets empty and the buffers are always available to pop and fill the data for the playback. But this has exposed an existing bug [gstreamer/-/issues/912](https://gitlab.freedesktop.org/gstreamer/gstreamer/-/issues/912) and a new issue where the `acquire_buffer`, yet again, gets blocked failing to dequeue the buffers and this time it is because one of the buffers is marked busy by the receiver (capture side) and that `process` method is not called until this particular buffer is cleared by the receiver.

I have been trying to get hold of this problem by checking where the lock is and why the sender and receiver are blocked, but it is taking a good number of hours of staring the code and the logs back and forth to understand the flow of the pipewire code.