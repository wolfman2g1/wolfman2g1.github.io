---
title: Home labbin' IT 5 - Moving to Freenas
date: 2018-03-30 12:00:00 -500
categories: [homelab,youtube]
layout: post
tags: [home labbin' it] 
youtubeId: tC7vbaHVxEM
---
{% include youtubePlayer.html id=page.youtubeId %}

This week we are talking about migrating from UNRAID to Freenas. Let's start with the why. For about a year I've been using UNRAID as my storage/VM/docker platform and it's been awesome. However like it says in the title, UNRAID is not raid, which has some good benefits and some pretty significant downsides. Because it doesn't use RAID it means I can mix an match drive capacities at will without issue and can connect a drive to any computer that can read XFS have full access to the data. Downside is it's not RAID. Because it's really just a software managed JBOD it means that you pay some penalties, one big one is that it's as fast as your slowest drive, and second because all the drives to don't share the task of a given read or write you get pretty slow performance. You can mitigate this somewhat by adding a cache drive, typically an SSD, which accepts all the writes and runs a "mover job" once a day to write the data to the larger capacity spinning drives. But what happens if the cache drive fails? The answer is you lose all the data that was still resident on the cache, you can of course use multiple cache drives in a RAID 1 which would help a lot but still I don't like this approach.

I briefly looked at  Rockstor, which on the surface seems legit, relatively new great interface and an awesome plug-in system. What kept me from using Rockstor is it uses BTRFS. Using BTRFS isn't an issue for me since it's basically designed to the Linux equivalent of ZFS. The show stopper was that it currently doesn't support RAID 5 or 6. It has support for it but it's not recommended for production use because of a writ-hole bug. A write-hole basically is this, you save a piece of data to your array, the OS makes the necessary syscall to have the data committed to disk, the call is returned as successful, except nothing was actually written. Seems like a bad look when I'm depending on this system to store my data.

I could have also done Xpenology but looking around at that eco-system it seems like it's more effort that it's worth.
So I settled on Freenas, yes Freenas is a bit long in the tooth but it works, and works well, for storage at least, the other features such as virtualization work ok but they aren't really top notch. Don't even get me started on docker support. But because I had tons of RAM and an extra 256GB SSD laying around it worked out great. I get good performance because of the way ZFS works and having the SSD to work as a L2 Cache ( L2ARC) has been great.