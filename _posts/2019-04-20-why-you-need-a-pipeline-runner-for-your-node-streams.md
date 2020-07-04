---
layout: post
title:  "Why You Need a Pipeline Runner For Your Node Streams"
date:   2019-04-20 09:12:00 +0000
comments: true
tags: 
 - node
 - streams
---

# Basic Node Streams

A basic Node streams application would look like this:

```javascript
const fs = require('fs')
const zlib = require('zlib')

fs.createReadStream(inputPath)
.pipe(zlib.createGzip())
.pipe(fs.createWriteStream(outputPath))
```

The key thing is that each stream provides a `pipe()` method, which is used to bolt on additional streams.

However, although this is correct, it doesn't lend itself to easy manipulation; in particular we have to start with one stream and bolt other streams on to it, which throws out the symmetry and makes the whole process difficult to reason about.

# A Better Model: A Pipeline of Streams

A better metaphor is of a pipeline that comprises multiple steps, each of which is a stream. In this model there is nothing 'special' about the first or last streams. Since version 10, Node has provided the module function [stream.pipeline()](https://nodejs.org/api/stream.html#stream_stream_pipeline_streams_callback) that does exactly this. This function can be promisified, which makes the code even easier to read. Our previous pipeline would now look like this:

```javascript
const stream = require('stream')
const util = require('util')
const pipeline = util.promisify(stream.pipeline)

const fs = require('fs')
const zlib = require('zlib')

await pipeline(
  fs.createReadStream(inputPath),
  zlib.createGzip(),
  fs.createWriteStream(outputPath)
)
```

Now it's much clearer that our streams are 'steps' in a pipeline, and they no longer play the role of *driving* the pipeline.

We'll use this pattern as our canonical pipeline because it is much easier to augment.

# Separating Pipeline Creation From Running

Now that we have the idea of a pipeline as our primary concept, we can start to play with it. In particular we'll find it extremely useful if we can separate the step of *creating* a pipeline from the step of *running* it.

We could then, for example, create a pipeline and check it for errors, even before it is run; we could check that the first step is a readable stream and the last step a writable one, for example, or ensure that all necessary resources such as databases or search engines are available before the pipeline starts.

We could also insert extra steps that wouldn't need to be specified explicitly in the pipeline, such as pushing and popping items to and from a queue between each step, or adding handlers for step-specific errors.

Another benefit of separating pipeline creation from execution is that we could optimise the pipeline before it's run, perhaps by merging or reordering steps. We might even create the pipeline on one machine, and send the pipeline to be run on another machine...or multiple machines if we have a way to split the work.

And not only could we do interesting things with the full pipeline, but we could even manipulate *parts* of the pipeline; we could run half of the pipeline, then materialise the results, and later run the rest of the pipeline using the saved results (incidentally allowing a pipeline to be upgraded, even whilst running), or we could execute individual pipeline steps as serverless functions, with all of the advantages that that would bring.

To get ourselves to a situation where these kinds of things are possible our next step is to separate the pipeline itself from its runner.

## Creating a Runner

The basic form of a runner is to first create some steps:

```javascript
const steps = [
  fs.createReadStream(inputPath),
  zlib.createGzip(),
  fs.createWriteStream(outputPath)
]
```

and then to run those steps:

```javascript
await pipeline(...steps)
```

If we wrap this functionality in a class then we have a foundation on which to add the other features we mentioned earlier. So let's create a class that allows us to *register* a set of steps with one method, and to *run* those steps with another method:

```javascript
class Runner {
  constructor() {
    const stream = require('stream')
    const util = require('util')
    this.pipeline = util.promisify(stream.pipeline)
  }

  register(steps) {
    this.steps = steps
    return this
  }

  run() {
    return this.pipeline(...this.steps)
  }
}
```

Now we can use our runner like this:

```javascript
const runner = new Runner()
const steps = [
  fs.createReadStream(inputPath),
  zlib.createGzip(),
  fs.createWriteStream(outputPath)
]

await runner
.register(steps)
.run()
```

It's exactly the same as our previous pipeline but with this structure we've got lots of points at which we can insert the new kinds of functionality that we mentioned earlier, by inheriting from the base class.

For example, we could add more features by way of the `register()` method, such as pipeline validation, step optimisation, insertion of additional steps, and so on. (And manipulation of the pipeline is as simple as manipulating an array.)

And we could modify the `run()` method to send the steps on to some other server, or launch more workers, sharing the work between them.

In future posts we'll start to flesh out how to achieve some of these more advanced features by building on this class.

# Conclusion

Node streams are incredibly powerful, and are a fundamental building-block of efficient and fast data-processing and transformation applications. However, the usual model of an input stream that pipes its output to another stream is quite difficult to manipulate and reason about. Node supports a better model by way of the `pipeline()` function, and this post shows how it can be used to construct pipelines that comprise multiple streams, and then to separately execute those pipelines.