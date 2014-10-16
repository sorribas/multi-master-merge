# multi-master-merge

A document database with multi master replication and merge support
based on [leveldb](https://github.com/rvagg/node-levelup), [fwdb](https://github.com/substack/fwdb) and [scuttleup](https://github.com/mafintosh/scuttleup)

```
npm install multi-master-merge
```

## Usage

``` js
var mmm = require('multi-master-merge')
var level = require('level')

var mdb = mmm(level('data.db'))

mdb.put('hello', {hello:'world'}, function(err, doc) {
  console.log('Inserted:', doc)
  mdb.get('hello', function(err, docs) {
    console.log('"hello" contains the following:', docs)
  })
})
```

When you do `mdb.get(key, cb)` you will always get an array of documents back.
The reason for this is to support multi master replication which means
that we might have multiple values for a given key.

## Replication

To replicate your database simply open a `sync` stream and pipe it to another
database

``` js
// a and b are two database instances
var s1 = a.sync() // open a sync stream for db a
var s2 = b.sync() // open a sync stream for db b

s1.pipe(s2).pipe(s1) // pipe them together to start replicating
```

Updates will now be replicated between the two instances.
If two databases inserts a document on the same key both of them will be
present if you do a `mdb.get(key)`

``` js
a.put('hello', {hello:'a'})
b.put('hello', {hello:'b'})

setTimeout(function() {
  a.get('hello', function(err, docs) {
    console.log(docs) // will print [{hello:'a'}, {hello:'b'}]
  })
}, 1000) // wait a bit for the inserts to replicate
```

## Merging

To combine multiple documents into a single one use `mdb.merge(key, docs, newDoc)`
If we consider the above replication scenario we have two documents for the key `hello`

``` js
a.get('hello', function(err, docs) {
  console.log(docs) // will print [{hello:'a'}, {hello:'b'}]
})
```

To merge them into a single document do

``` js
a.get('hello', function(err, docs) {
  a.merge('hello', docs, {hello:'a + b'})
})
```

Merges will replicate as well

``` js
// wait a bit for a to replicate to b
b.get('hello', function(err, docs) {
  console.log(docs) // will print [{hello:'a + b'}]
})
```

## License

MIT