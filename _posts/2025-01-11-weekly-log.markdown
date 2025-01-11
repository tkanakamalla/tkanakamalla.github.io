---
layout: post
title:  "GstPipeWire - buffers and deviceproviders"
date:   2025-01-11
categories: weekly-log
---

### Picking up from the previous week
Added a new patch to the MR [!2228](https://gitlab.freedesktop.org/pipewire/pipewire/-/merge_requests/2228) as per suggestions from Wim to undo a dequeue by returning the unused buffer to the front of the `dequeued` instead of the pushing it to the end of the `dequeued` or `queued`. So the primary issue reported in the [#1326](https://gitlab.freedesktop.org/pipewire/pipewire/-/issues/1326) is more or less addressed, but the other use cases involving the seek event like trick modes (fast-forward/rewind), seek in paused state etc., are still posing issues.

Another issue I was looking at the [previous week]({% post_url 2025-01-04-weekly-log %}), where the playback client  (`pipewiresink` as a `provider`) and the capture client (`pipewiresrc` for e.g.,) get blocked upon a seek/flush event in the the playback side, is still unresolved. Both the clients keep waiting for their respective `process` callback to be triggered. The root cause of this issue is still not known but there is a new finding related to this issue. The playback client is usually the `driver` when running as a `provider` but when issue occurs, it is not `driving` anymore as per the `pw-top`. So this could be the reason the `process` callback is not triggered and hence the buffers are not flowing. The logs still suggest that it is driving mode so I have hit a dead end in this issue for time being.

One more observation from the logs, the `pipewiresink` fails to dequeue a buffer just before it gets blocked and the reason for failure of the dequeue is, the `pw_buffer` was marked busy. The current explanation for this is, the corresponding `GstBuffer` pushed downstream by `pipewiresrc` for consumption and that is not released yet.

### How is a `pw_buffer` related to a `GstBuffer`?
Amid all this hotchpotch of buffers queueing/dequeueing/unqueueing, it could be quite confusing about how the buffers flow, what the lifecycle of a buffer is in the PipeWire and how it is handled by the GstPipeWire elements. Let me try to explain that with a little more details.

A pipewire client when it creates and connects a new `pw_stream`([Stream](https://docs.pipewire.org/page_streams.html)), it gets allocated a negotiated number of the `pw_buffer`s. The [add_buffer](https://docs.pipewire.org/structpw__stream__events.html#abd09d7c76b0e14a201a438f09e609b29) can be used to get a callback whenever a new `pw_buffer` is created.

In the `gstpipewire` elements, a `GstBufferPool` is used for buffer management. In the `add_buffer` event callback for each `pw_buffer`, a new `GstBuffer` a created and mapped to it, allocating a `GstMemory` instance for each data segment in `spa_buffer->datas`. These components and related bufferpool metadata are stored in the `pw_buffer`'s `user_data`.

For any `pw_stream`, the [process](https://docs.pipewire.org/structpw__stream__events.html#a512bd219ce44ba1a688d600773efe84b) event is triggered whenever the buffers are ready to be consumed/produced by the [data loop](https://docs.pipewire.org/data-loop_8h.html) thread. Upon the `process` event, the playback client should dequeue a `pw_buffer`, fill it with the produced data and enqueue it for the playback. Similarly, the capture client should dequeue the `pw_buffer` and enqueue it back after consuming the data so that it can be recycled for the next use.

In case of the `pipewiresink` i.e., a producer client, the upstream elements pick each `GstBuffer` from the bufferpool for filling the data that is to be rendered. The `GstBufferPool's` [acquire_buffer](https://gstreamer.freedesktop.org/documentation/gstreamer/gstbufferpool.html?gi-language=c#gst_buffer_pool_acquire_buffer) method is used to dequeue a `pw_buffer` from the `pw_stream`, pick the associated `GstBuffer` and return that to fill the data. This `GstBuffer` is passed to `render` method of the `pipewiresink`. The render function will then retrive the corresponding `pw_buffer` and push it to the stream's queue for playback.

In case of the flush event, the `GstBuffer` is dropped without rendering so buffers are not enqueued and hence they are won't be available for dequeue in the subsequent `acquire_buffer` calls and the queue gets empty eventually. This is what I have addressed in the [!2228](https://gitlab.freedesktop.org/pipewire/pipewire/-/merge_requests/2228)

### GstPipeWireDeviceProvider
Another thing I was working on this week.

Looking at the flaky state of the pipewire elements, it makes sense to list `pulsesink/src` instead of `pipewiresink/src` for audio, at least, and continue to have the `pipewiresrc` for video devices by the Pipewire's device provider. For this, I had to dig a little bit into the concept of a `GstDeviceProvider` this week. The `GstPipeWire` plugin has device provider implementation and it currently overrides, the `pulsedeviceprovider` for alsa devices, `v4l2deviceprovider` for v4l2 devices and `libcameradeviceprovider` for `libcamera` devices, with the `pipewiredeviceprovider` elements.

Since the devices in a provider are listed together, it seemed not possible to prioritize `pipewiresrc` just for video but deprioritize them for audio. So we might need to go with the option of remove the hiding of `pulsedeviceprovider` and not adding the `sink/src` elements for audio devices to the `pipewiredeviceprovider`.