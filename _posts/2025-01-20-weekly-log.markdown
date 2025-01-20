---
layout: post
title:  "GstPipeWire - better handling during the seek event"
date:   2025-01-20
categories: weekly-log
---

Last week, I was looking into two items

1) Changes in the `pipewiredeviceprovider` to [expose pulse devices and exclude pipewire elements for audio](https://gitlab.freedesktop.org/tkanakamalla/pipewire/-/commit/e990e2777d932211146d724b003548c6372613bc). An MR needs to be raised for this followed by any improvements/changes based on the review

2) Fixing the issues around seek/flush events in the `pipewiresink`. Primarly I was trying to fix the failure to resume playback after seek in PAUSED state. This issue was because, the `pipewirepool` was set to flushing when the `pipewiresink` is set to PAUSED, if there was a seek event at this point, the `acquire_buffer` of the `pipewirepool` was returning `GST_FLOW_FLUSING` and this prevents the buffer flow. Hence pipeline was held in the PAUSED state forever, unable to transition to PLAYING state. I have made the changes to [avoid flush of pipewirepool in PLAYING_TO_PAUSED](https://gitlab.freedesktop.org/pipewire/pipewire/-/merge_requests/2247) to fix this.

However this was leading to an occasional deadlock situation while setting the pipewiresink to NULL. This happens particularly if the upstream element is using its own bufferpool instead of the `pipewirepool`. The `acquire_buffer` gets called from the `_render` method of the pipewiresink if the buffer does not belong to the `pipewirepool` and so if we don't flush the `pipewirepool` (the `flush_start` method of the `pipewirepool` does a `signal`, so the conditional_wait gets unblocked) while transitioning from PLAYING->PAUSED, the `_render` could get blocked on a conditional variable, unaware of its state transition. To fix this, Wim pushed a couple of commits [c7ccc5ab](https://gitlab.freedesktop.org/pipewire/pipewire/-/commitc7ccc5abcaf4404be9ef8f9926b9073f90b91d2e) and [d36a8677](https://gitlab.freedesktop.org/pipewire/pipewire/-/commit/d36a867788bf64c8b53983dedcbead49ff759494). With that the seek events seem to be handled better but this needs some more testing this week to be sure enough.

I also spent a little time exploring the possibities of getting trick modes working with `pipewiresink` but that requires more thought and planning, so moved that for later.