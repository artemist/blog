---
layout: post
title: On the Politics of Time
date: 2021-06-27
---

# What is a Time Zone, anyway?
The tzdata database, as you'll see on many Linux systems, says a time zone is an area where the same offset has been observed since 1970.
Each time zone has an identifier based on the largest city in that location. For example, `America/New_York` refers to the time observed
in NYC and is also used in various other cities, such as Washington DC. However, I think this is a fundamental misunderstanding of what a
time zone actually is.

A time zone isn't a place. It's people. Generally, many people in an area can agree what the time is, but that's not always true.
For many computer users and server operators it's always UTC. It doesn't matter where you are, but UTC is the "canonical time".

If you're in Palestine or Xinjiang then your ethnic group likely determines what time you think it is. Palestineans in the West Bank may follow
daylight savings changes set by the PLO and no the Israeli government. On the other hand, Israelis living in illegal settlements will use
DST set by the Israeli government. The DST changeovers can differ by up to a few days, meaning two people can be "right" about the correct time
at a certain place while differing by an hour.
In Xinjiang, some Uyghurs may use Xinjiang Time (UTC+6) while many Han Chinese use Beijing time (UTC+8) for all their activities.

A less politically charged example: At Michigania (a camp run by the University of Michigan Alumni Association) the time is moved back by one
hour for a portion of the week. While you're surrounded by people who think the time is UTC-4, you think the time is UTC-5.


# Time is Political
This whole post was brought about because of a decision to remove 
