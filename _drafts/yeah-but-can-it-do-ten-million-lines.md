---
layout: post
title: Yeah...but can it do ten million lines?
author: Will Vaughn
---

Here at Cascade Energy, the dev team is all hands on deck, engineering a new pipeline for receiving energy data to
allow our mechanical engineers, energy managements specialists, and clients to work with their
data in meaningful ways within SENSEI. We're using a bevy of Amazon Web Services to architect this system 
in the cloud which we hope will be maintainable, scalable, and nimble in ways our current data pipeline is not.

One facet of this new architecture necessitates breaking large files into pieces about 100 
lines in length to be passed along a series of AWS Lambda functions. Breaking these large files into pieces is
helps manage the cost of [AWS Lambda](http://aws.amazon.com/lambda/), because it is billed based on time to run, 
and memory consumed. The payoff? Completely horizontally scalable data processing without any infrastructure management.
It's a good thing, such a good thing!

This is the part of the "Digest" system, as we call it, that I've been working on recently. Before we can send
data along a series of Lambda functions, we needed to build a tool that did the "chunking" of large files. We
call it the "digest-chunker", and we chose to implement it in [NodeJS](https://nodejs.org/). This is how it went.
 
#### Minimal Implementation

I wanted it to pick up a csv file stored in [Amazon S3](http://aws.amazon.com/s3/), streams it through a csv parser 
(`$ npm install csv`), take 100 lines, and put them in a message to pass along. I decided to develop and test with a
file I thought was fairly large in size, 25,000 lines of data. Not the top end of file size our clients and energy
management team members would like to work with, but it's nothing to slouch at either. I had a working version in a 
few days, and was pretty pleased with myself. I ran a demo of it to the rest of the team to prove that I hadn't just 
been phoning it in. To which their reply was:

> Yeah...But, can it do a ten million line file?

#### Anticipating Real Usage

Working within an engineering culture is a blessing. That is, if you like having everything you build used in a capacity
100x beyond what you ever intended or thought possible! So when my team came at me with this reply, I knew what they
were saying. They were trying to make sure we achieved durability when it came time for our users to go to town 
with it.

> I don't know. I mean, yeah probably. Well, maybe. Fine, I'll write a tool to build a ten million line csv...

Yeah. No. The answer is no, it can not do a ten million line file. It turned out that the 25,000 line sample I'd been
working with was just small enough for me to not notice an enormous problem with the fire hose that is NodeJS file I/O
streaming. The grossly over simplified flow of the "digest-chunker" is this:

1. Open a stream
2. Pipe that stream to a parser
3. Count to 100 as things come out of that parsed stream.
4. Send an event to SNS (This is an asynchronous operation)
5. Keep going until the file is done.

The unexpected problem was Step 2's incredible overwhelming speed (the fire hose), and Step 4's comparative slowness.
The read events from the stream would stack up in the NodeJS event queue, and essentially block chunks from sending 
to SNS. It's like if you were heading into the line at Starbucks, turned around for a second to see if you forgot to 
turn your car lights off, and when you turned around there were 300,000 people ordering flat whites and spinach 
feta wraps.

Just as troubling -- node streams feed data into an internal buffer. Any file of significant size will 
eventually fill up that buffer, and grind node to a crawl.

#### The "Use-Case" vs. "Churn" scales

At some point when you encounter a barrier like this you have to decide (I'll say "decide" but I mean "guess") whether
or not you can accommodate a use case in a matter of time which does not negate the utility in terms
of cost. We chose to take this one on, for a few reasons:

(I am switching from the pronoun "I", to the pronoun "we" from here on in this blog because from this point
on "I" had no idea what to do)

1. We were curious, and we needed to know how node I/O streams
work because they're a foundational part of our plan to receive energy data.
2. We didn't know where the limit was for when a file would be too large.

We desired to engineer a system that didn't care about how large a file was, because it would never be working with a 
portion of that file which was too large for it to handle.

In essence, we had to get real meta and start chunking our chunking! So we did.

#### Kink the fire hose.

We used the pipe/unpipe methods of node streams to turn on/off the flow of data from the open csv file.
To decide when to kink and un-kink the hose, we needed to track the number of pending SNS events we had active at any
given time.

The three components below broadly describe our solution to the problem.

*chunkManager.js*

Responsibilities:

- Count lines, send an SNS event every time you reach 100
- Increment how many "active sends" exist everytime send a chunk of 100 lines.
- Decrement that count every time an "active send" completes.
- Constantly notify a *chunkMediator* of the "active send" count anytime it changes.
    
*chunkMediator.js*

Responsibilities:

- Relay the "active send" count between the *chunkMediator* and the *streamManager*
    
*streamManager.js*

Responsibilities:

- Store a reference to the open csv file stream from s3.
- Store a reference to the csv parser stream that file is piped into.
- Listen to the *chunkMediator* and respond anytime the "active send" count changes
- Kink the hose anytime the "active send" count reaches 25.
- Unkink the hose once the "active send" count gets back below 25.
    
There's not a lot of detail there, and the devil is definitely in the details. But if you'd like to know
more feel free to @mention me on twitter [@nackjicholsonn](https://twitter.com/nackjicholsonn). I'd be glad to
explain it further, share more of the code, or have to completely throw it away because you bring me another
solution. This is by no means put out as THE solution to this problem, just the solution we were able to come to in the 
time we allotted to solving it. That's software development, and that's our job.
