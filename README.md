# streamx

An iteration of the Node.js core streams that fixes tons of issues whilst maintaining most backwards compat.

```
npm install @mafintosh/streamx
```

## Main improvements from Node.js core stream

#### Proper lifecycle support.

Streams have an `_open` function that is called before anything else and a `_destroy`
function that is always run as the last part of the stream.

This makes it easy to maintain state.

#### Easy error handling

Fully integrates a `.destroy()` function. When called the stream will wait for any
pending operation to finish and call the stream destroy logic.

Close is *always* the last event emitted and `destroy` is always ran.

#### `pipe()` error handles

`pipe` accepts a callback that is called when the pipeline is fully drained.
It also error handles the streams provided and destroys both streams if either
of them fail.

#### No more object mode

Instead a `map` function can be provided to map your input data into buffers
or other formats. To indicate how much buffer space each data item takes
an `byteLength` function can be provided as well.

#### Simplicity

This is a full rewrite, all contained in one file.

Lots of stream methods are simplified based on how I and devs I work with actually use streams in the wild.

## Usage

``` js
const { Readable } = require('@mafintosh/streamx')

const rs = new Readable({
  read (cb) {
    this.push('Cool data')
    cb(null)
  }
})

rs.on('data', data => console.log('data:', data))
```

## API

This streamx package contains 4 streams similar to Node.js core.

## Readable Stream

#### `rs = new stream.Readable([options])`

Create a new readable stream.

Options include:

```
{
  highWaterMark: 16384, // max buffer size in bytes
  map: (data) => data, // optional function to map input data
  byteLength: (data) => size // optional function that calculates the byte size of input data
}
```

In addition you can pass the `open`, `read`, and `destroy` functions as shorthands in
the constructor instead of overwrite the methods below.

The default byteLength function returns the byte length of buffers and `1024`
for any other object. This means the buffer will contain around 16 non buffers
or buffers worth 16kb when full if the defaults are used.

#### `rs._read(cb)`

This function is called when the stream wants you to push new data.
Overwrite this and add your own read logic.
You should call the callback when you are fully done with the read.

Can also be set using `options.read` in the constructor.

Note that this function differs from Node.js streams in that it takes
the "read finished" callback.

#### `drained = rs.push(data)`

Push new data to the stream. Returns true if the buffer is not full
and you should push more data if you can.

If you call `rs.push(null)` you signal to the stream that no more
data will be pushed and that you want to end the stream.

#### `data = rs.read()`

Read a piece of data from the stream buffer. If the buffer is currently empty
`null` will be returned and you should wait for `readable` to be emitted before
trying again. If the stream has been ended it will also return `null`.

Note that this method differs from Node.js streams in that it does not accept
an optional amounts of bytes to consume.

#### `rs.unshift(data)`

Add a piece of data to the front of the buffer. Use this if you read too much
data using the `rs.read()` function.

#### `rs._open(cb)`

This function is called once before the first read is issued. Use this function
to implement your own open logic.

Can also be set using `options.open` in the constructor.

#### `rs._destroy(cb)`

This function is called just before the stream is fully destroyed. You should
use this to implement whatever teardown logic you need. The final part of the
stream life cycle is always to call destroy itself so this function will always
be called wheather or not the stream ends gracefully or forcefully.

Can also be set using `options.destroy` in the constructor.

Note that the `_destroy` might be called without the open function being called
in case no read was ever performed on the stream.

#### `rs.destroy([error])`

Forcefully destroy the stream. Will call `_destroy` as soon as all pending reads have finished.
Once the stream is fully destroyed `close` will be emitted.

If you pass an error this error will be emitted just before `close` is, signifying a reason
as to why this stream was destroyed.

#### `rs.pause()`

Pauses the stream. You will only need to call this if you want to pause a resumed stream.

#### `rs.resume()`

Will start consuming the stream as fast possible.

#### `writableStream = rs.pipe(writableStream, [callback])`

Efficently pipe the readable stream to a writable stream (can be Node.js core stream or a stream from this package).
If you provide a callback the callback is called when the pipeline has fully finished with an optional error in case
it failed.

To cancel the pipeline destroy either of the streams.

#### `rs.on('readable')`

Emitted when data is pushed to the stream if the buffer was previously empty.

#### `rs.on('data', data)`

Emitted when data is being read from the stream. If you attach a data handler you are implicitly resuming the stream.

#### `rs.on('end')`

Emitted when the readable stream has ended and no data is left in it's buffer.

#### `rs.on('close')`

Emitted when the readable stream has fully closed (i.e. it's destroy function has completed)

#### `rs.on('error', err)`

Emitted if any of the stream operations fail with an error. `close` is always emitted right after this.

#### `rs.destroyed`

Boolean property indicating wheather or not this stream has been destroyed.

## Writable Stream

#### `ws = new stream.Writable([options])`

Create a new writable stream.

Options include:

```
{
  highWaterMark: 16384, // max buffer size in bytes
  map: (data) => data, // optional function to map input data
  byteLength: (data) => size // optional function that calculates the byte size of input data
}
```

In addition you can pass the `open`, `write`, `flush`, and `destroy` functions as shorthands in
the constructor instead of overwrite the methods below.

The default byteLength function returns the byte length of buffers and `1024`
for any other object. This means the buffer will contain around 16 non buffers
or buffers worth 16kb when full if the defaults are used.

#### `ws._open(cb)`

This function is called once before the first write is issued. Use this function
to implement your own open logic.

Can also be set using `options.open` in the constructor.

#### `ws._destroy(cb)`

This function is called just before the stream is fully destroyed. You should
use this to implement whatever teardown logic you need. The final part of the
stream life cycle is always to call destroy itself so this function will always
be called wheather or not the stream ends gracefully or forcefully.

Can also be set using `options.destroy` in the constructor.

Note that the `_destroy` might be called without the open function being called
in case no write was ever performed on the stream.

#### `ws.destroy([error])`

Forcefully destroy the stream. Will call `_destroy` as soon as all pending reads have finished.
Once the stream is fully destroyed `close` will be emitted.

If you pass an error this error will be emitted just before `close` is, signifying a reason
as to why this stream was destroyed.

#### `drained = ws.write(data)`

Write a piece of data to the stream. Returns `true` if the stream buffer is not full and you
should keep writing to it if you can. If `false` is returned the stream will emit `drain`
once it's buffer is fully drained.

#### `ws._write(data, callback)`

This function is called when the stream want to write some data. Use this to implement your own
write logic. When done call the callback and the stream will call it again if more data exists in the buffer.

Can also be set using `options.write` in the constructor.

#### `ws.end()`

Gracefully end the writable stream. Call this when you no longer want to write to the stream.

Once all writes have been fully drained `finish` will be emitted.

#### `ws._flush(callback)`

This function is called just before `finish` is emitted, i.e. when all writes have flushed but `ws.end()`
have been called. Use this to implement any logic that should happen after all writes but before finish.

Can also be set using `options.flush` in the constructor.

#### `ws.on('finish')`

Emitted when the stream has been ended and all writes have been drained.

#### `ws.on('close')`

Emitted when the readable stream has fully closed (i.e. it's destroy function has completed)

#### `ws.on('error', err)`

Emitted if any of the stream operations fail with an error. `close` is always emitted right after this.

#### `ws.destroyed`

Boolean property indicating wheather or not this stream has been destroyed.

## Duplex Stream

#### `s = new stream.Duplex([options])`

A duplex stream is a stream that is both readable and writable.

Since JS does not support multiple inheritance it inherits directly from Readable
but implements the Writable API as well.

If you want to provide only a map function for the readable side use `mapReadable` instead.
If you want to provide only a byteLength function for the readable side use `byteLengthReadable` instead.

Same goes for the writable side but using `mapWritable` and `byteLengthWritable` instead.

## Transform Stream

#### `ts = new stream.Transform([options])`

A transform stream is a duplex stream that maps the data written to it and emits that as readable data.

Has the same options as a duplex stream except you can provide a `transform` function also.

#### `ts._transform(data, callback)`

Transform the incoming data. Call `callback(null, mappedData)` or use `ts.push(mappedData)` to
return data to the readable side of the stream.

Per default the transform function just remits the incoming data making it act as a pass-through stream.

## Contributing

If you want to help contribute to streamx a good way to start is to help writing more test
cases, compatibility tests, documentation, or performance benchmarks.

If in doubt open an issue :)

## License

MIT
