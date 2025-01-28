---
layout: post
title:  "GstPipewire - just a few updates"
date:   2025-01-25
categories: weekly-log
---
I have continued investigation of the deadlock occurring during seek/pause operations when the sink is [driving](https://docs.pipewire.org/page_streams.html#sec_stream_driving) (provide mode). Given the issue's rare occurrence (limited to provide mode) and complexity, I've decided to temporarily deprioritize it to focus on more pressing matters.

I have briefly checked [GstPipewireSrc deadlocks when pipeline state changes from PLAYING to NULL](https://gitlab.freedesktop.org/pipewire/pipewire/-/issues/4511). However, debugging progress has been limited since the issue isn't consistently reproducible.

Currently, my main focus is on addressing the issue - [RFC: Don't provide bufferpool for audio](https://gitlab.freedesktop.org/pipewire/pipewire/-/issues/4519). I've submitted a draft merge request [2259](https://gitlab.freedesktop.org/pipewire/pipewire/-/merge_requests/2259) addressing this. Before marking it ready for review, I need to thoroughly test additional corner cases this week.