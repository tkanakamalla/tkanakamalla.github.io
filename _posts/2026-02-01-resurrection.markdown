---
layout: post
title:  "Hello, again"
date:   2026-02-01
categories: Hello, weekly-log
---

Yay, I am writing a new post after a loooooong break (almost an year later). I really hope I will be able to finish this and post it before it gets pre-empted yet again by my parental duties which has been one of the main <s>excuses</s> reasons I quote myself for failing to invest time on any other things.

I will save all my <s>rants</s> experiences about being a parent for an exclusive post. But may be I am just way too worried dealing with those unscheduled tantrums and emotional meltdowns of a toddler. Or, I am just being unrealistic in expecting to be productive at work despite sleep disruptions every night.  Anyway, in the last few days it feels like things are getting a bit easier and I am starting to find some "extra" time to do something productive. Hence, this attempt to resume journaling.

So, catching up on the happenings, I moved to [Centricular][1] around mid of last year since [Asymptotic has paused operations][2]. Nothing changed in the nature of the work though. I still work remotely on multimedia (mostly GStreamer) based projects but with a bigger group of people now.

Some important updates summarizing the work I have done since my last post:

- Few [clock related fixes][10] in the `pipewiresink` before wrapping up at Asymptotic. I was hoping to write a past on my learnings during this work but I just could make manage time for that. But I want to take it up sooner or later. It will also be a good chance to revisit `PipeWire` and get back some context so I can continue contributing to it again.

- Worked on GStreamer FLV plugins to add multitrack audio/video support following the new specification of Enhanced RTMP(v2). I wrote about this in the [Centricular devlog][3] and also [spoke][4] at the last year's [GStreamer Conference at London][5].

- Wrote a Rust based [GIF decoder][6] element using the [Decoder in the gif crate][7]. I learnt quite a bit dealing with the transparency and background colour in the images apart from the understanding how Gif encoding works. I am hoping to write about that in more detail in a separate post "very soon".

- Got the WHEP [Client][8] and [Server][9] signaller implementations over the finish line finally. The basic functionality is up-to-date including the server-side-offer handling in the client as per the draft specification.

- Fixed some big and small bugs in the `rswebrtc` plugins and particularly in the last few days, worked on adding the `request` type src pads to the `webrtcsrc` element. The request pads can be used as a way to control the media types that we want to filter in the remote offer from the WebRTC producer or propose as in the local offer to the producer. [Here][11] is the related merge request.

Overall, it has been a busy year juggling between parental duties and work. But I want to do more writing and be regular in posting things. I also want to make time to stay outdoors a little more and work on my fitness. Funny, I am thinking of new year resolutions a month late.

[1]: https://centricular.com/
[2]: https://asymptotic.io/blog/indefinite-hiatus/
[3]: https://centricular.com/devlog/2025-11/Multitrack-Audio-Capability-in-FLV/
[4]: https://gstconf.ubicast.tv/videos/the-road-to-enhanced-flv-and-rtmp-in-gstreamer_xgl8aiouqk/
[5]: https://gstreamer.freedesktop.org/conference/2025/
[6]: https://gstreamer.freedesktop.org/documentation/gif/gifdec.html?gi-language=c
[7]: https://docs.rs/gif/latest/gif/struct.Decoder.html
[8]: https://gstreamer.freedesktop.org/documentation/rswebrtc/whepclientsrc.html?gi-language=c
[9]: https://gstreamer.freedesktop.org/documentation/rswebrtc/whepserversink.html?gi-language=c
[10]: https://gitlab.freedesktop.org/pipewire/pipewire/-/merge_requests/2337
[11]: https://gitlab.freedesktop.org/gstreamer/gst-plugins-rs/-/merge_requests/2796
