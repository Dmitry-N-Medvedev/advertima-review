# Overview

## The Task

Read through the solution and provide us in a simple document a brief explanation of your understanding of the purpose of the code. In addition, we would like to hear a minimum of 3 improvement suggestions. Please outline the benefits and potential of your suggestions.

## Purpose of the code

Getting binary messages from C++ module and somehow processing it.
It would be easier to understand it all should there business requirements exist.
Without them all that is left to me is to humbly guess which purpose this Node module serves for.

## Suggestions

### Data flow

Since there are several modules residing in the same machine and exchanging data using websockets
on local interface for communication between them would take at least two context switches
for each packet, which slows down the whole thing.

In the first draft of this document I wanted to share with you an idea that unix sockets should be used instead of websockets, since they are way faster especially when paired with memory mapped file. Further reflections of mine lead me to the thought that inter module communications on the edge machine should be implemented via some kind of a message bus. At lease I would seek to employ a message bus, since it allows for decoupling of functionality with all the benefits of such decoupling.

Taking into consideration that the edge machine is quite a powerful one, I would suggest we should use Redis database with at least one extra module installed - [Redis Streams](https://redis.io/topics/streams-intro). This would allow us to design the whole system aligned with CQRS ideas and implement proper event sourcing.
Should Redis Streams be employed, we could go further and simplify part of our code removing some libraries. I would start with removing the RxJS and rewriting the code base using EventEmitters. This would definitely reduce the amount of JavaScript code in memory that would save some hundreds bytes of it and millions of CPU ticks. Should need arise, I would love to recommend the [redis-fast-driver](https://github.com/h0x91b/redis-fast-driver) library which is actually written in C++ with a tiny JavaScript wrapper around it. I have been using it for a couple of years now and have had next to no problems either with it, or with its author.

NB: please have a look at the [RedisEdge](https://redislabs.com/solutions/redisedge/) and [RedisAI](https://redislabs.com/blog/redis-ai-first-steps/) modules since they might further greatly simplify the whole system.

### Binary messages vs JSON

A lot of effort has been put into defining structures and their validation:

1. every time via `enum`:
   1. [WSConnectionStatus](src/connection/WSConnection.ts)
   2. [RPCRecordType, RPCFunction, RPCCommands, RPCResponseSubject, BinaryDataType](src/constants/Constants.ts)
   3. [BinaryType](src/types/index.ts)
2. some times via `interface`s and `class`es:
   1. [RecognitionMetadata](src/model/person-detection/PersonDetection.ts)
   2. [PlayoutEventOptions, ExtendedPlayoutEventOptions, PlayoutEvent, StartEvent, EndEvent](src/model/playout-event/PlayoutEvent.ts)
   3. [PersonOptions, ContentOptions, FlushOptions](src/poi/test-utils/common.ts)
   4. [BinaryMessage, JSONMessage](src/types/index.ts)
   5. etc.
3. copying an object (actually a `struct`) as in [PersonDetectionMessage.fromObject](src/messages/person-detection/PersonDetectionMessage.ts) via [lodash/cloneDeep](https://lodash.com/docs/4.17.11#cloneDeep)

In all these cases `enum`s, `class`es, and `interface`s denote structures.
Most of them seem (apart from [ContentMessage](src/messages/content/ContentMessage.ts), [PersonDetectionMessage](src/messages/person-detection/PersonDetectionMessage.ts),and [PersonFlushMessage](src/messages/person-flush/PersonFlushMessage.ts)) lack proper validation checks.

Further comes a sketchy list of what I would do if I were given a right to introduce changes to the system as I see fit:

1. I would definitely redefine all of these structures (or at least only those that leave boundaries of a module) in a form of [FlatBuffers IDL](https://google.github.io/flatbuffers/md__schemas.html) and have the `flatc` compiler generate corresponding C++ and JavaScript/TypeScript implementations. This would allow for the following benefits:
   1. strict type checking and validation, since the FlatBuffers support type system.
   2. uselessness of JSON and its validation, thanks to the said type system. Getting rid of JSON would save huge amount of CPU cycles and several hundreds of bytes of RAM. During the development phase it seems like a natural approach to use JSON as the medium, since a lot gets changed in the code base. As the project matures a bit and all the message formats (in our case) get settled down we would no longer want to regularly examine the contents of the messages "by hand". But since this point in time our system would still go on suffering from the bloated nature of JSON. Of course, employing FlatBuffers would require a certain degree of discipline regarding what we transmit via messages, but this would result in a way cleaner and, which is more important, performant system.
   3. TypeScript would become obsolete and the code base would drop quite a lot of hustles connected with code transpilation. The new code would be expressed mostly of simple classes and standalone functions. Very easy for understanding and testing.

NB: as far as I can understand the C++ module provides data in the form of a `msgpack` ([POISnapshot](src/poi/POISnapshot.ts)). Please see [benchmarks](https://google.github.io/flatbuffers/flatbuffers_benchmarks.html). If you are the author of the C++ module it might be not so time consuming a task to substitute `msgpack` for FlatBuffers to gain even more speed.

### Architecture

I cannot emphasize it more, but I would suggest we should redesign the whole edge system for it to become a bunch of tiny microservices talking to eash other via Redis. Let's say, initially they would be written for Node and managed by [PM2](https://github.com/Unitech/PM2/). Provided that each one does just one thing, it would be very easy for us later to rewrite at least some of the in a more performant programming language - Rust.

From what I have been given, I could not see how the whole system is deployed onto an edge machine. Still, I presume there must be some sort of engine (like Kubernetes) supporting it. We could make our system more secure if we implemented it in Rust and run it within a special operating system designed from scratch for containers. I am sure that security is a critical part for our success.

### Project structure

It is simple. Please consider using a monorepo. I have pleasant experience using [Lerna](https://github.com/lerna/lerna). This would greatly simplify inter service dependencies.

### Logging

Instead of using console, I would recommend using Redis Streams. Its easier, more robust and performant approach. On the other side of Redis there might be a tiny log transmission service sitting and injesting the log stream and sending it, for instance, down to the cloud for further processing.

### CI/CD

This is yet to be figured out. How do you update your software on the edge devices?

### To Sum Up a bit

1. move away from JSON to the strictly typed FlatBuffers
   1. this will reduce RAM requirements;
   2. this will speed up processing messages;
2. move away from websockets on local interface but employ a message bus such as Redis (with appropriate modules)
   1. offload data manipulation tasks to C++ code base of Redis - it processes data way faster and is capable of handling way more data than JavaScript;
   2. it will give you real asynchrony;
3. move away from TypeScript (yes it's a hype, I know) since your code base will be comprised of very tiny services (functions mostly and no longer than a couple of pages of code) whose types will be strictly controlled by FlatBuffers
   1. less software on your `devDependencies` list => less bugs and dependencies, less CPU cycles
4. cut your modules into pieces so that each one does just one type of data conversion
   1. this would allow you later to gradually rewrite your JavaScript code in, say, Rust
