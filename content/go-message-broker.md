+++
title = "Writing a distributed message broker in Go"
description = "Exploring the internals of how message brokers work and my thoughts on 'Grokking streaming' book."
draft = false
date = "2022-07-14"
template = "page.html"
menu = "sandesh"
+++
> This post is a first of a part of series. For all posts in this series, check the [index](https://example.org) page.

I recently came across Manning's [Grokking Streaming Systems](https://www.manning.com/books/grokking-streaming-systems) and after completing at least the first half of the book, it did not seem very complicated. Indeed, working at a [SIEM company](https://logpoint.com) as a Solutions Engineer really inures you to the nitty gritty details of how large-scale streaming systems handle workloads effectively, but one particular thing that caught my eye in every design was the message queue. In the book, each service has an incoming and outgoing queue except the terminal services, like producers and consumers. If we decouple the queue from the system, we will, effectively be left with something that can be reused, but does not make sense in isolation. For e.g. a text-processing streaming system can "consume" events and maybe convert the text into ascii, but if there's no connection between the two, then there's not much use to the streaming system. Another issue with having a privatized queue on each component is that if the component fails, all data in RAM is lost. Indeed, this can be avoided by using disk-based persistence for each component, but where does it end!

So an integral part of every streaming system is the message broker - something like [Kafka](https://google.com/?q=Kafka+Broker). My work uses kafka internally as well, and so this blog post is dedicated to creating a _trivial_ example of such a message broker. Others have shown efforts towards this direction like [Benthos](https://benthos.dev) and Yuriy Nasretdinov's [Chukcha](https://github.com/YuriyNasretdinov/chukcha) (he even [live streams](https://www.youtube.com/watch?v=t3FdULDRfRM&list=PLWwSgbaBp9XqeuIuTWqpNtvf_EL0I4TJ2) about this!). Ours will be yet-another-kafka clone so this project will be called "Yak" for lack of a better name (and a creative name-er).

Let's get started with the design

<br />

## Design and Philosophy

Kafka uses a log-based file to write contents internally instead of keeping them in memory. The power of simple sequential disk write 