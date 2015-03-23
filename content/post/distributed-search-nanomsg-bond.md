+++
Categories = ["Distributed Systems"]
Description = "Distributed Search Engine with Nanomsg and Bond"
Tags = ["Cxx", "Nanomsg", "Bond"]
date = "2015-03-22T13:05:45+01:00"
title = "Distributed Search Engine with Nanomsg and Bond"

+++

Exploring Microsoft's open source Bond framework by building a distributed search engine.
I'm using bond for serialization/deserialization and nanomsg for communication.

The source for this C++14 project is located at: https://github.com/daniel-j-h/DistributedSearch


## Motivation

A few weeks ago Microsoft open sourced Bond, a cross-platform framework for serialization/deserialization, similar to Google's Protocol Buffers or Cap'n Proto. I have some experience with Cap'n Proto in particular, so this weekend I wanted to give Bond a try, since I had a few hours to spare.

### A Distributed Search Engine

Rob Pike introduced Go's concurrency patterns with an example of a Google-inspired search.
The slides [are still available](https://talks.golang.org/2012/concurrency.slide#42) (please quickly skim slides 42-52) and the talk is also on Youtube. Let's pick up this idea and implement it!

The approach taken is roughly as follows:

* Query multiple services concurrently: for web results, video results, news and so on
* Gather the results, merge and show them to the user
* Replicate the services and query the replicas, too
* Do not wait forever, set timeouts

This is the general idea. For more information please see the talk mentioned above.

Now this project consists of the following. The communication part, for sending requests and receiving responses. I'm using nanomsg for this.
But first we have to serialize/deserialize our requests, i.e. the keyword to search for and the matches we receive from the services. I'm using Bond for this.


## Nanomsg

[nanomsg](http://nanomsg.org/) is a communication library, designed to provide you with patterns, such as Pub/Sub, Req/Rep, the Survey pattern and more.
You may be familiar with ZeroMQ, nanomsg is more or less the same with a few exceptions. I'm using nanomsg since I'm already familiar with it.

Now we design our distributed search engine as follows: a Search service issues user-provided queries concurrently against the WebSearch service, the VideoSearch service and so on. The results are then merged and shown to the user. For this we're using nanomsg's [Survey pattern](http://nanomsg.org/v0.4/nn_survey.7.html).
The Survey pattern sends messages to multiple locations and gathers the responses. For our simple project this is good enough, but having a single Surveyor is not the best idea and you probably want to think about how to factor this into the design.

### Surveyor

With the Survey pattern the so called Surveyor (our user-facing Search service) has to [bind to the endpoint](https://github.com/daniel-j-h/DistributedSearch/blob/62a84caa478b3421e275d57ad0311c879ff89b51/Service.h#L28-L37), on which so called Respondents connect to.
The Surveyor also sets a timeout after which the survey is over and subsequent results coming in are discarded for this particular survey.

For every user query the Surveyor now does the following.
Initially [broadcast the request to the Respondents](https://github.com/daniel-j-h/DistributedSearch/blob/62a84caa478b3421e275d57ad0311c879ff89b51/Service.h#L57-L58).
Then [gather all results from the Respondents](https://github.com/daniel-j-h/DistributedSearch/blob/62a84caa478b3421e275d57ad0311c879ff89b51/Service.h#L70) as long as the timeout has not expired.

### Respondents

Respondents (our WebSearch service, ImageSearch service, and so on) [connect to the endpoint](https://github.com/daniel-j-h/DistributedSearch/blob/62a84caa478b3421e275d57ad0311c879ff89b51/Service.h#L100-L104), indicating they want to participate in surveys.
For this the services spin in an eventloop, waiting for requests to process.
Once they [receive a request](https://github.com/daniel-j-h/DistributedSearch/blob/62a84caa478b3421e275d57ad0311c879ff89b51/Service.h#L121-L122) they handle it (i.e. they search for results) and [send matches for this query back](https://github.com/daniel-j-h/DistributedSearch/blob/62a84caa478b3421e275d57ad0311c879ff89b51/Service.h#L147-L148).

Now that the basic communication is set up, let's do the serialization/deserialization part.


## Bond

Using [Microsoft's Bond framework](https://github.com/Microsoft/bond), we're able to serialize and deserialize our messages (i.e. the keyword to search for and the matches we receive) before sending them over the wire.
For this we [define our messages](https://github.com/daniel-j-h/DistributedSearch/blob/62a84caa478b3421e275d57ad0311c879ff89b51/Messages.bond) in a .bond schema.
The bond schema compiler now [lets us generate stubs](https://github.com/daniel-j-h/DistributedSearch/blob/62a84caa478b3421e275d57ad0311c879ff89b51/Makefile#L5-L6) from this schema and they are included in the source repository.
You probably want to augment the messages with more information, such as timestamps, ratings, and so on. For this project a simple schema is good enough.

What's interesting now is the fact that the bond compiler is also able to spit out Python and C# stubs, which should make it possible to implement the Surveyor and Respondents in other languages, too. But I did not try this, yet.

### Serialization

Now what the Surveyor (our user-facing search service) does is to [serialize the user-provided keyword](https://github.com/daniel-j-h/DistributedSearch/blob/62a84caa478b3421e275d57ad0311c879ff89b51/Service.h#L47-L54) before handing it over to nanomsg.
The Respondents (our WebSearch service, ImageSearch service, and so on) also [serialize their results](https://github.com/daniel-j-h/DistributedSearch/blob/62a84caa478b3421e275d57ad0311c879ff89b51/Service.h#L138-L145) before sending them back to us.


### Deserialization

For every query the Surveyor sends it has to [deserialize the responses](https://github.com/daniel-j-h/DistributedSearch/blob/62a84caa478b3421e275d57ad0311c879ff89b51/Service.h#L76-L80) nanomsg hands us during the survey.
The respondents similarly have to first [deserialize the keyword](https://github.com/daniel-j-h/DistributedSearch/blob/62a84caa478b3421e275d57ad0311c879ff89b51/Service.h#L129-L133) the Surveyor sends us.

The types we used in the schema now integrate perfectly into the language. Therefore [merging responses](https://github.com/daniel-j-h/DistributedSearch/blob/62a84caa478b3421e275d57ad0311c879ff89b51/Service.h#L82-L83) on the Surveyor side is quite easy, using set semantics. [Populating](https://github.com/daniel-j-h/DistributedSearch/blob/62a84caa478b3421e275d57ad0311c879ff89b51/Service.h#L139) the data structure with responses on the Respondent's side can be done in the same way.

Both serialization and deserialization are quite easy with Bond. Especially the (bytes, size)-tuple handling as required when interacting with nanomsg is not as bad as it was with Cap'n Proto.
Fortunately Kenton Varda improved the Cap'n Proto library in this regard, after [a discussion on the mailinglist](https://groups.google.com/forum/#!msg/capnproto/viZXnQ5iN50/B-hSgZ1yLWUJ).


## Putting It All Together

With the serialization/deserialization and communication part done, all we have to do is put the pieces together.
That is, wrap what we built and provide a few ways of customization.

The user-facing Search service [interacts with the user and queries the services](https://github.com/daniel-j-h/DistributedSearch/blob/62a84caa478b3421e275d57ad0311c879ff89b51/Search.cc#L10-L20).
The search services [wait for queries and handle them](https://github.com/daniel-j-h/DistributedSearch/blob/62a84caa478b3421e275d57ad0311c879ff89b51/WebSearch.cc#L9-L20) by sending dummy results for now.
Now let's take a look at what we just built!

Spin up the user-facing Search service and try issuing queries against it:

    ./Search

```bash
Search>
How many horns does a unicorn have?

Results>
```

No results. Right, we do not have any search service running. Let's spin up a few:

    ./WebSearch
    ./VideoSearch

```bash
Search>
How many horns does a unicorn have?

Results>
 * First Video Result
 * First Web Result
 * Second Video Result
 * Second Web Result
```

Great! We get results back from those two services, without even noticing the connection establishment and communication done in the background during which our program was active at all time.

Now what makes this even more awesome is that nanomsg guarantees us a handful of nice properties.
For example, our user-facing Search service does not care about what services are currently available.
You are also able to disconnect or re-connect any service at any time and the user only sees this in the results available.
This also allows us to start up e.g. multiple WebSearch service replicas in case some are too slow to respond within the timout.
Finally, nanomsg also [handles auto-reconnects](http://nanomsg.org/v0.1/nn_setsockopt.3.html).

Furthermore we do not depend on the transport used. Check this out for a local IPC engine:

    ./Search "ipc:///tmp/search.ipc"
    ./WebSearch "ipc:///tmp/search.ipc"



## Summary

In building a distributed search engine you hopefully learnt something about communication and serialization.
Using nanomsg and its Survey pattern makes the communication part quite easy.
Bond makes the serialization and deserialization part simple to implement.

The source is hosted on GitHub: https://github.com/daniel-j-h/DistributedSearch
