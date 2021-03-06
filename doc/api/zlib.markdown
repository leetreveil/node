## Zlib

You can access this module with:

    var zlib = require('zlib');

This provides bindings to Gzip/Gunzip, Deflate/Inflate, and
DeflateRaw/InflateRaw classes.  Each class takes the same options, and
is a readable/writable Stream.

### Examples

Compressing or decompressing a file can be done by piping an
fs.ReadStream into a zlib stream, then into an fs.WriteStream.

    var gzip = zlib.createGzip();
    var fs = require('fs');
    var inp = fs.createReadStream('input.txt');
    var out = fs.createWriteStream('input.txt.gz');

    inp.pipe(gzip).pipe(out);

To use this module in an HTTP client or server, use the
[accept-encoding](http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.3)
on requests, and the
[content-encoding](http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.11)
header on responses.

**Note: these examples are drastically simplified to show
the basic concept.**  Zlib encoding can be expensive, and the results
ought to be cached.  See <a href="#memory_Usage_Tuning">Memory Usage
Tuning</a> below for more information on the speed/memory/compression
tradeoffs involved in zlib usage.

    // client request example
    var zlib = require('zlib');
    var http = require('http');
    var fs = require('fs');
    var request = http.get({ host: 'izs.me',
                             path: '/',
                             port: 80,
                             headers: { 'accept-encoding': 'gzip,deflate' } });
    request.on('response', function(response) {
      var output = fs.createWriteStream('izs.me_index.html');

      switch (response.headers['content-encoding']) {
        // or, just use zlib.createUnzip() to handle both cases
        case 'gzip':
          response.pipe(zlib.createGunzip()).pipe(output);
          break;
        case 'deflate':
          response.pipe(zlib.createInflate()).pipe(output);
          break;
        default:
          response.pipe(output);
          break;
      }
    });

    // server example
    // Running a gzip operation on every request is quite expensive.
    // It would be much more efficient to cache the compressed buffer.
    var zlib = require('zlib');
    var http = require('http');
    var fs = require('fs');
    http.createServer(function(request, response) {
      var raw = fs.createReadStream('index.html');
      var acceptEncoding = request.headers['accept-encoding'];
      if (!acceptEncoding) {
        acceptEncoding = '';
      }

      // Note: this is not a conformant accept-encoding parser.
      // See http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.3
      if (acceptEncoding.match(/\bdeflate\b/)) {
        response.writeHead(200, { 'content-encoding': 'deflate' });
        raw.pipe(zlib.createDeflate()).pipe(response);
      } else if (acceptEncoding.match(/\bgzip\b/)) {
        response.writeHead(200, { 'content-encoding': 'gzip' });
        raw.pipe(zlib.createGzip()).pipe(response);
      } else {
        response.writeHead(200, {});
        raw.pipe(response);
      }
    }).listen(1337);

### Constants

All of the constants defined in zlib.h are also defined on
`require('zlib')`.  They are described in more detail in the zlib
documentation.  See <http://zlib.net/manual.html#Constants>
for more details.

### zlib.createGzip([options])

Returns a new [Gzip](#zlib.Gzip) object with an [options](#options).

### zlib.createGunzip([options])

Returns a new [Gunzip](#zlib.Gunzip) object with an [options](#options).

### zlib.createDeflate([options])

Returns a new [Deflate](#zlib.Deflate) object with an [options](#options).

### zlib.createInflate([options])

Returns a new [Inflate](#zlib.Inflate) object with an [options](#options).

### zlib.createDeflateRaw([options])

Returns a new [DeflateRaw](#zlib.DeflateRaw) object with an [options](#options).

### zlib.createInflateRaw([options])

Returns a new [InflateRaw](#zlib.InflateRaw) object with an [options](#options).

### zlib.createUnzip([options])

Returns a new [Unzip](#zlib.Unzip) object with an [options](#options).


### zlib.Gzip

Compress data using gzip.

### zlib.Gunzip

Decompress a gzip stream.

### zlib.Deflate

Compress data using deflate.

### zlib.Inflate

Decompress a deflate stream.

### zlib.DeflateRaw

Compress data using deflate, and do not append a zlib header.

### zlib.InflateRaw

Decompress a raw deflate stream.

### zlib.Unzip

Decompress either a Gzip- or Deflate-compressed stream by auto-detecting
the header.

### Options

Each class takes an options object.  All options are optional.

Note that some options are only
relevant when compressing, and are ignored by the decompression classes.

* chunkSize (default: 16*1024)
* windowBits
* level (compression only)
* memLevel (compression only)
* strategy (compression only)

See the description of `deflateInit2` and `inflateInit2` at
<http://zlib.net/manual.html#Advanced> for more information on these.

### Memory Usage Tuning

From `zlib/zconf.h`, modified to node's usage:

The memory requirements for deflate are (in bytes):

    (1 << (windowBits+2)) +  (1 << (memLevel+9))

that is: 128K for windowBits=15  +  128K for memLevel = 8
(default values) plus a few kilobytes for small objects.

For example, if you want to reduce
the default memory requirements from 256K to 128K, set the options to:

    { windowBits: 14, memLevel: 7 }

Of course this will generally degrade compression (there's no free lunch).

The memory requirements for inflate are (in bytes)

    1 << windowBits

that is, 32K for windowBits=15 (default value) plus a few kilobytes
for small objects.

This is in addition to a single internal output slab buffer of size
`chunkSize`, which defaults to 16K.

The speed of zlib compression is affected most dramatically by the
`level` setting.  A higher level will result in better compression, but
will take longer to complete.  A lower level will result in less
compression, but will be much faster.

In general, greater memory usage options will mean that node has to make
fewer calls to zlib, since it'll be able to process more data in a
single `write` operation.  So, this is another factor that affects the
speed, at the cost of memory usage.
