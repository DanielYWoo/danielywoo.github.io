---
layout: post
title:  "Develop Applications with Time - Time Scale"
date:   2021-02-08 10:00:00
published: false
---

# Overview

In the last article we discussed <b>floating time</b> and <b>explicit time</b>, they are practically used in applications need localization and globalization. In this article, we will discuss some interesting topics:

- Time Scale
  - Solar Time
  - Atomic Time
- Leap Second
- Wall Clock & Logical Clock

# Time Scale

## Solar Time
Before we talk about time scale, let me ask you a question, how to measure time? Time is meaningful only when there is something moving, so we measure time by Sun moves, the earliest time scale is **day**. But that's not accurate enough for rocket launching. Let's see why.

A day is actually a **Solar Day**, which is the internal we see the Sun in the highest position from the Earth.  A solar day is not accurate because of precession, the quote below is from wikipedia:

*Earth's rotation is not a simple rotation around an axis that would always remain parallel to itself. Earth's rotational axis itself rotates about a second axis, [orthogonal](https://en.wikipedia.org/wiki/Orthogonality) to Earth's orbit, taking about 25,800 years to perform a complete rotation. This phenomenon is called the [precession of the equinoxes](https://en.wikipedia.org/wiki/Axial_precession).*

So if we devide a solar day into 24 hours, or 86400 seconds a day, each second is changing, most likely longer. The Earth's tidal friction lengthens the day by 2.3 ms/century. The 2004 Indian Ocean earthquake is thought to have shortened it by 2.68 microseconds. Scientists have done a lot of work (radio telescopes, pulsar timing etc) to make it accurate and finally defined UT1 time scale . Since UT1 still based on Earth movement, each second will be non-even.

## Atomic Time
If we want to launch a rocket, we cannot use UT1 since the second is non-even. We need a fixed second, that's why Atomic Time Scale is designed.
International Atomic Time (TAI, from the French name temps atomique international) is a man-made, laboratory timescale. Its units is the SI second (standard international second). One SI second is the time that elapses during 9,192,631,770 (9.192631770 x 10 9 ) cycles of the radiation produced by the transition between two levels of the cesium 133 atom. To make it more accurate, TAI is weighted average of the time kept by BIMP over 400 atomic clocks in 50+ labs. So, it never adjust by Sun/Earth/Moon/Tide position and rotation speed.
A similar time scale system is GPS time. Due to speed difference, GPS time is 7 micro seconds slower than a clock at sea level, 45 micro seconds faster due to gravity. The US Navy Observatory adjust it constantly and resets every 19.6 years. GPS satellites have built-in atomic clocks but they are just for backup, only get used in case US Navy Observatory cannot send commands to adjust the clocks in GPS satellites.

## UTC
Atomic Time is great for science but it's not aligned with our calendar which is based on solar days, we cannot use it in our daily life. For that reason, UTC is introduced.

UTC is a compromise, stepping with SI seconds but periodically reset by a leap second to match UT1. IERS (IERS - International Earth Rotation and Reference Systems Service) ensures the difference between the UTC and UT1 never exceeds 0.9 seconds and periodically announce leap seconds to the public.

## Time Scale Examples
Let's take a look at the example of TAI, UT1 and UTC in comparision. The table below shows three consequtive moments. TAI time is always increasing monotonically and evenly, UT1 does not get aligned with TAI and you see there is a 0.407 fraction. UTC is kind of compromise between them, 

TAI=2009-01-01T00:00:32.0; UTC=2008-12-31T23:59:59.0; UT1≅2008-12-31T23:59:58.407
TAI=2009-01-01T00:00:33.0; UTC=2008-12-31T23:59:60.0; UT1≅2008-12-31T23:59:59.407
TAI=2009-01-01T00:00:34.0; UTC=2009-01-01T00:00:00.0; UT1≅2009-01-01T00:00:00.407



Unix Epoch Time: is UTC but does not keep leap seconds! Epoch time is *similar* to UTC.
The most common approach is to simply step the clock back by one second when the clock gets to 00:00:00 UTC. This is implemented in the Linux kernel and it is enabled by default. BUT, this is dangerous to applications depending Epoch time. Use nptd –x to disable it.
BTW, don’t sign a digital certificate at 2008-12-31T23:59:60

