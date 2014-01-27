---
layout: post
category : Blog
tagline: "Really hard!"
tags : [AI, Code]
---
{% include JB/setup %}

It's probably just a lack of experience with behaviour trees talking here, but working with them for the first time seems really quite difficult. Here's an example tree from the game we're working on.

This is supposed to drive this big guy.

He has a bunch of different behaviours. He can attack you with two different hand attacks, or with two different beam attacks that fire from his belly eye. He can collapse after a sustained beating and go into a vulnerable state for you to drop a building on his head, or he can be stunned. He can be reacting to having his hands attacked, or he can be reacting to having his eyeball attacked. While he's not stunned or collapsed, his eyeball can be either opened or closed. While he's stunned, his eyeball is always opened. Attacking his eye while it's open reduces his health, and while it's closed just plays a twitch animation and does no damage. That's probably everything:)

There's a couple of patterns that have started to crop up in this behaviour tree and some others that we've worked on. Firstly, at the top level of the tree, we use a dynamic priority list composite to dynamically react to things that are happening in the world. 