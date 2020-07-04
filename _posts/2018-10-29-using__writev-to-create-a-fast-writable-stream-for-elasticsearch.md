---
layout: post
title:  "Using _writev() to Create a Fast, Writable Stream for Elasticsearch"
date:   2018-10-29 11:37:01 +0000
comments: true
tags: 
 - elasticsearch
 - node
 - streams
---

We all know how great Node streams are. But it wasn't until I recently needed to create (yet another) writable stream wrapper for Elasticsearch that I realised just how much work the streaming APIs can do for you. And in particular how powerful the `_writev()` method is.

<!--snip-->

I was looking to wrap the Elasticsearch client in a writable stream so that I could use it in a streaming pipeline. I've done this many times before, in many different contexts--such as creating Elasticsearch modules to be used with Gulp and Vinyl--so I was all set to follow the usual pattern:

* my first step would be to set up an Elasticsearch client, using the Elasticsearch API;
* I'd then add a function that gets called with whatever entry should be written to the Elasticsearch server;
* to speed writing up I wouldn't write this entry straight to the server, but instead buffer each of the entries in an array (the size of which would of course be configurable). Then, once the buffer was full the entries would be written en masse to the Elasticsearch server using the [bulk update API](https://www.elastic.co/guide/en/elasticsearch/client/javascript-api/current/api-reference.html#api-bulk) (which is much, much faster than writing records one at a time);
* when the source of the data for the writable stream indicates that there is no more data to send I'd check whether there is any data still in the buffer, and if so call a 'flush' function;
* and once all data is flushed, I'd delete the client.

None of this will probably surprise you, and you'd no doubt write an interface to Elasticsearch in much the same way yourself.

But what might surprise you--especially if you haven't looked at [Node's Writable Streams](https://nodejs.org/api/stream.html#stream_writable_streams) for a while--is how many of these steps could be done for you by the Node libraries.

To kick things off, let's create a class that extends the [Node stream `Writable` class](https://nodejs.org/api/stream.html#stream_class_stream_writable):

```javascript
const stream = require('stream')

class ElasticsearchWritableStream extends stream.Writable {
}

module.exports = ElasticsearchWritableStream
```

Now we can start adding each of the features in our list.

# Creating an Elasticsearch Client

The first step we described above was to create an Elasticsearch client, using the [Elasticsearch API](https://www.npmjs.com/package/elasticsearch), so let's add that to the constructor of our class:

```javascript
const stream = require('stream')
const elasticsearch = require('elasticsearch')

class ElasticsearchWritableStream extends stream.Writable {
  constructor(config) {
    super()
    this.config = config
    
    /**
     * Create the Elasticsearch client:
     */

    this.client = new elasticsearch.Client({
      host: this.config.host
    })
  }
}

module.exports = ElasticsearchWritableStream
```

We can now call our class with some configuration, and we'll have a writable stream with an Elasticsearch client:

```javascript
const sink = new ElasticsearchWriteStream({ host: 'es:9200' })
```

Of course, this stream doesn't do anything yet, so let's add the method that the streaming infrastructure will call whenever some other stream wants to write a record.

# Writing Records

When implementing a writable stream class, the only method we need to provide is [`_write()`](https://nodejs.org/api/stream.html#stream_writable_write_chunk_encoding_callback_1) which is called whenever new data is available from the stream that is providing that data. In the case of our Elasticsearch stream, to forward the record on we only need to call [`index()`](https://www.elastic.co/guide/en/elasticsearch/client/javascript-api/current/api-reference.html#api-index) on the client that we created in the constructor:

```javascript
class ElasticsearchWritableStream extends stream.Writable {
  constructor(config) {
    ...
  }
  
  /**
   * When writing a single record, we use the index() method of
   * the ES API:
   */

  async _write(body, enc, next) {

    /**
     * Push the object to ES and indicate that we are ready for the next one.
     * Be sure to propagate any errors:
     */

    try {
      await this.client.index({
        index: this.config.index,
        type: this.config.type,
        body
      })
      next()
    } catch(err) {
      next(err)
    }
  }
}
```

Note that once we've successfully written our record we then call `next()` to indicate to the streaming infrastructure that we're happy to receive more records, i.e., more calls to `_write()`. In fact, if we *don't* call `next()` we won't receive any more data.

# Index and Type

When writing to Elasticsearch we need to provide the name of an index and a type for the document, so we've added those to the config that was provided to the constructor, and we can then pass these values on to the call to `index()`. We'll now need to invoke our stream with something like this:

```javascript
const sink = new ElasticsearchWriteStream({
  host: 'es:9200',
  index: 'my-index',
  type: 'my-type'
})
```

# Buffering

As things stand, we already have a working writable stream for Elasticsearch. However, if we're planning to insert hundreds of thousands of records then it will be slow, and a simple optimisation would be to buffer the records and use the [bulk update API](https://www.elastic.co/guide/en/elasticsearch/client/javascript-api/current/api-reference.html#api-bulk).

## Bulk Update API

The bulk update API allows us to perform many operations at the same time, perhaps inserting thousands of records in one go. Rather than defining each record to be inserted as we did with the `index()` call, we need to create [a list that contains pairs of entries](https://www.elastic.co/guide/en/elasticsearch/reference/6.3/docs-bulk.html); one that indicates the operation to carry out--such as an insert or update--and one that contains the data for the operation.

## Using an Array

The usual 'go to' implementation here would be to create an array in the class constructor, and then push the rows of data into that array with each call to `_write()`. Then, when the array is full, construct a call to the bulk API, still within the `_write()` method.

The problem here though, is that in order to properly implement backpressure we need quite a sophisticated interaction with the `next()` function; we need to allow data to flow to our stream as long as the buffer is not full, and we need to prevent new data arriving until we've had a chance to write the records to Elasticsearch.

It turns out that the Node streaming API can manage the buffer *and* the backpressure for us.

## _writev()

Although the bare minimum we need to provide in our writable stream class is a `_write()` method, there is another method we can create if we like, called [`_writev()`](https://nodejs.org/api/stream.html#stream_writable_writev_chunks_callback). Where the first function is called once per record, the second is called with a *list* of records. In a sense, the streaming API is doing the whole *create an array and store the items until the array is full and then send them on* bit for us.

Here's what our `_writev()` method would look like:

```javascript
class ElasticsearchWritableStream extends stream.Writable {
  ...

  async _writev(chunks, next) {
    const body = chunks
    .map(chunk => chunk.chunk)
    .reduce((arr, obj) => {
      /**
       * Each entry to the bulk API comprises an instruction (like 'index'
       * or 'delete') on one line, and then some data on the next line:
       */
      
      arr.push({ index: { } })
      arr.push(obj)
      return arr
    }, [])

    /**
     * Push the array of actions to ES and indicate that we are ready
     * for more data. Be sure to propagate any errors:
     */

    try {
      await this.client.bulk({
        index: this.config.index,
        type: this.config.type,
        body
      })
      next()
    } catch(err) {
      next(err)
    }
  }
}
```

The streaming API will buffer records and then at a certain point hand them all over to our `_writev()` function. This gives us the main benefit of buffering data--that we can then use the bulk update API--without actually having to create and manage a buffer, or look after backpressure.

# Buffer Size

If we'd created the buffer ourselves we'd have had complete control over how big the buffer is, but can we still control the buffer size if the Node streaming API is managing the buffer for us?

It turns out we can, by using the [generic `highWaterMark` feature](https://nodejs.org/api/stream.html#stream_constructor_new_stream_writable_options), which is used throughout the streams API to indicate how large buffers should be.

The best way to implement this in our writable stream is to have two parameters for our constructor:

* one which will provide configuration for the Elasticsearch connection, such as server address, timeout configuration, the name of the index and type, and so on;
* another which provides settings for the writable stream itself, such as `highWaterMark`.

This is easily added, like so:

```javascript
class ElasticsearchWritableStream extends stream.Writable {
  constructor(config, options) {
    super(options)
    this.config = config
    
    /**
     * Create the Elasticsearch client:
     */

    this.client = new elasticsearch.Client({
      host: this.config.host
    })
  }
  
  ...
}
```

And now we can control the size of the buffer--and hence, the number of records that are being written by each call to the bulk API--by setting options in the constructor:

```javascript
const esConfig = {
  host: 'es:9200',
  index: 'my-index',
  type: 'my-type'
}
const sink = new ElasticsearchWriteStream(
  esConfig,
  { highWatermark: 1000 }
)
```

# Closing the Elasticsearch Client

All that remains from our original checklist is to close the client when there is no more data to receive. To implement this, all we need to do is to add another optional method, [`_destroy()`](https://nodejs.org/api/stream.html#stream_writable_destroy_err_callback). This is called by the streaming infrastructure when there is no more data, and would look something like this:

```javascript
_destroy() {
  return this.client.close()
}
```

# Conclusion

As you can see, the Node streaming API has done much of the work of buffering, for us, which means that we don't get bogged down with trying to implement backpressure properly. By providing us with the methods `_write()`, `_writev()` and `_destroy()` our code ends up very clean, and focuses our attention on only the parts required to spin up and destroy a connection to Elasticsearch, and the functions required to write a single record, or a batch. The full implementation looks like this:

```javascript
const stream = require('stream')
const elasticsearch = require('elasticsearch')

class ElasticsearchWritableStream extends stream.Writable {
  constructor(config, options) {
    super(options)
    this.config = config
    
    /**
     * Create the Elasticsearch client:
     */

    this.client = new elasticsearch.Client({
      host: this.config.host
    })
  }
  
  _destroy() {
    return this.client.close()
  }

  /**
   * When writing a single record, we use the index() method of
   * the ES API:
   */

  async _write(body, enc, next) {

    /**
     * Push the object to ES and indicate that we are ready for the next one.
     * Be sure to propagate any errors:
     */

    try {
      await this.client.index({
        index: this.config.index,
        type: this.config.type,
        body
      })
      next()
    } catch(err) {
      next(err)
    }
  }

  async _writev(chunks, next) {
    const body = chunks
    .map(chunk => chunk.chunk)
    .reduce((arr, obj) => {
      /**
       * Each entry to the bulk API comprises an instruction (like 'index'
       * or 'delete') and some data:
       */
      
      arr.push({ index: { } })
      arr.push(obj)
      return arr
    }, [])

    /**
     * Push the array of actions to ES and indicate that we are ready
     * for more data. Be sure to propagate any errors:
     */

    try {
      await this.client.bulk({
        index: this.config.index,
        type: this.config.type,
        body
      })
      next()
    } catch(err) {
      next(err)
    }
  }
}

module.exports = ElasticsearchWritableStream
```