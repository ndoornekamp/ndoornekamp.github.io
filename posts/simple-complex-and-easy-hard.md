---
title: Simple vs. complex (and easy vs. hard)
date: 2026-05-01
permalink: /simple-vs-complex
---

Complex originates from 'complected', which means 'to be braided together': something made up of multiple, _intertwined_ parts. It is the opposite of 'simplex', which means 'consisting of only _one part_'.

Note how this definition is independent of 'difficulty': something can be hard to do or understand but consist of one part, whereas something can be easy while consisting of many parts. In the context of software engineering, it is often easy to install a library, but doing so _complects_ your project with that library. In fact, quite often choosing the easy option increases complexity. Or alternatively: it is hard to do keep things simple.

A current example is the complection of almost all large companies with (US-based) cloud providers. Unease about this complection is increasing, but very few companies can afford the migration to an on-premise solution. Running your own hardware is hard and therefore costly, but simple in the sense that you are not intertwined with said cloud providers.

## Why should we care about simplicity?

As many people involved with building software will have experienced, development often slows down as a system grows. Starting out, new features are churned out daily, whereas months (or years, or decades) later it may take days (or weeks, or months) to change something that sounds small. The complexity of the system can greatly accelerate this phenomenon:

- To be able to change a system, we need to be able to reason about it: to understand how it works, how its parts interact and how it interacts with other systems. We need to keep a mental model of the system in our mind.
- We can only keep so many things (concepts, components, etc.) in our working memory.
- Things that are complected need to be considered _together_, because changing one of them may affect the others.

Therefore: the more complex the system you're working on is, the more quickly you will reach the limits of your ability to fit the mental model in your working memory. Once this limit is exceeded, the probability that a change comes with unintended consequences skyrockets. It's not a hard limit though - as you become more familiar with the system, its parts and their interactions will become more intuitive and take up less space in your working memory. However, the time that is required to reach this familiarity is a big factor in the common slow-down of development as a system grows.

## Further reading

- ["Simple Made Easy" by Rich Hickey](https://www.infoq.com/presentations/Simple-Made-Easy/)
