+++
title = "Writing a message broker in Go"
description = "Exploring the internals of how message brokers work and my thoughts on 'Grokking streaming' book."
draft = false
date = "2022-07-14"
template = "page.html"
menu = "sandesh"
+++
> This post is a first of a part of series. For all posts in this series, check the [index](https://example.org) page.

## Introduction: 

I recently came across Manning's [Grokking Streaming Systems](https://www.manning.com/books/grokking-streaming-systems) and after completing at least the first half of the book, it did not seem very complicated. Indeed, working at a [SIEM company](https://logpoint.com) as a Solutions Engineer really inures you to the nitty gritty details of how large-scale streaming systems handle workloads effectively, but one particular thing that caught my eye in every design was the message queue. 

In design, each service in the streaming system has an incoming and outgoing queue and a processing part (except source and sink components). Each service/component reads from upstream outgoing queue and pushes data to its own downstream outgoing queue. For an example, let's consider a simple text-streaming system that takes in a stream of text, converts them to lowercase and sends them to a cloud bucket to be stored: 

<div class = "mermaid" style = "margin: 30px auto;" >
graph TD
    A["Source (like http)"] --> B["Transformer (Converts source text to lowercase)"]
    B --> C["AWS Bucket"]
</div>

Let's think of an implementation: 
1. Write a HTTP listener
2. Write the Transformer that pulls in data from a socket 
3. Write a sink that pulls in data from a socket and pushes it to AWS bucket. 

There are a couple of ways this can have issues: 
1. Using sockets means every service needs to know about every other service, or an intermediate glue service needs to be created that spawns service. For e.g. if I want multiple transformers, I will need to write a `TransformerSpawner` that tracks spawned transformers, their output ports and the upstream service(HTTP Listener)'s output port and effectively distribute data across spawned Transformers. **AND** if the upstream service fails, all the data in it's output socket are lost. (I am talking about failure here, not congestion)

2. If a code change needs to be made on a Transformer, all services need to hold their data to prevent the last case in our previous thought. If I push a new code, then HTTP Listener will keep data in its queue, and then release it once `TransformersSpawner` comes back online. One way to prevent this is to keep the data in a service-specific-persistent storage like a log file. This will also solve the data loss issue when a service fails, as it can just replay its internal log. but as the number of services grows, this can quickly become cumbersome with issues like `max number of open files`. 

Well, then, how do we solve this? By using a message broker. The message broker keeps all the messages, shards them across multiple nodes as required, persists them and returns them to the consumers on request. Not only this, a message broker gives you processing guarantees, like `At least once` or `Exactly Once` which we call delivery guarantees so that you don't have to worry about lost messages, duplicated messages, etc. So an integral part of every streaming system is the message broker - something like [Kafka](https://google.com/?q=Kafka+Broker). My work uses kafka internally as well, and so this blog post is dedicated to creating a _trivial_ example of such a message broker. Others have shown efforts towards this direction like [Benthos](https://benthos.dev) and Yuriy Nasretdinov's [Chukcha](https://github.com/YuriyNasretdinov/chukcha) (he even [live streams](https://www.youtube.com/watch?v=t3FdULDRfRM&list=PLWwSgbaBp9XqeuIuTWqpNtvf_EL0I4TJ2) about this!). Ours will be yet-another-kafka clone so this project will be called "Yak" for lack of a better name (and a creative namer).

Let's get started with the design

<br />

## Design and Philosophy

Kafka uses a log-based file to write contents internally instead of keeping them in memory. The power of simple sequential disk write 