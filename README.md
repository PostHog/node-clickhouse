Database interface for http://clickhouse.yandex
===

```
npm install @apla/clickhouse
```

[![travis](https://travis-ci.org/apla/node-clickhouse.svg)](https://travis-ci.org/apla/node-clickhouse)
[![codecov](https://codecov.io/gh/apla/node-clickhouse/branch/master/graph/badge.svg)](https://codecov.io/gh/apla/node-clickhouse)

Synopsis
---

```javascript
var ch = new ClickHouse ({host: clickhouse.host, port: 8123, auth: "user:password"});
// or
var ch = new ClickHouse (clickhouse.host);

// do the query, callback interface, not recommended for selects
ch.query ("CREATE DATABASE clickhouse_test", function (err, data) {

});

// promise interface (requires 'util.promisify' for node < 8, Promise shim for node < 4)
ch.querying ("CREATE DATABASE clickhouse_test").then (…);

// it is better to use stream interface to fetch select results
var stream = ch.query ("SELECT 1");

// or collect records yourself
var rows = [];

stream.on ('metadata', function (columns) {
  // do something with column list
});

stream.on ('data', function (row) {
  rows.push (row);
});

stream.on ('error', function (err) {
  // TODO: handler error
});

stream.on ('end', function () {
  // all rows are collected, let's verify count
  assert (rows.length === stream.supplemental.rows);
  // how many rows in result are set without windowing:
  console.log ('rows in result set', stream.supplemental.rows_before_limit_at_least);
});

```

API
---

### new ClickHouse (options)

```javascript
var options = {
  host: "clickhouse.msk",
  queryOptions: {
    profile: "web",
    database: "test"
  },
  omitFormat: false
};

var clickHouse = new ClickHouse (options);
```

If you provide options as a string, they are assumed as a host parameter in connection options

Connection options (accept all options documented
for [http.request](https://nodejs.org/api/http.html#http_http_request_options_callback)):

 * **auth**:     authentication as `user:password`, optional
 * **host**:     host to connect, can contain port name
 * **pathname**: pathname of ClickHouse server or `/` if omited,

`queryOptions` object can contain any option from Settings (docs:
[en](https://clickhouse.yandex/docs/en/operations/settings/index.html)
[ru](https://clickhouse.yandex/docs/ru/operations/settings/index.html)
)

For example:

 * **database**: default database name to lookup tables etc.
 * **profile**: settings profile to use
 * **readonly**: don't allow to change data
 * **max_rows_to_read**: self explanatory

Driver options:

 * **omitFormat**: `FORMAT JSONCompact` will be added by default to every query.
 You can change this behaviour by providing this option. In this case you should
 add `FORMAT JSONCompact` by yourself.
 * **syncParser**: collect data, then parse entire response. Should be faster, but for
 large datasets all your dataset goes into memory (actually, entire response + entire dataset).
 Default: `false`
 * **dataObjects**: use `FORMAT JSON` instead of `FORMAT JSONCompact` for output.
 By default (false), you'll receive array of values for each row. If you set dataObjects
 to true, every row will become an object with format: `{fieldName: fieldValue, …}`


### var stream = clickHouse.query (statement, [options], [callback])

Query sends a statement to a server

Stream is a regular nodejs object stream, it can be piped to process records.

Stream events:

 * **metadata**: when a column information is parsed,
 * **data**: when a row is available,
 * **error**: something is wrong,
 * **end**: when entire response is processed

After response is processed, you can read a supplemental response data, such as
row count via `stream.supplemental`.

Options are the same for `query` and `constructor` excluding connection.

Callback is optional and will be called upon completion with
a standard node `(error, result)` signature.

You should have at least one error handler listening. Via callbacks or via stream errors.
If you have callback and stream listener, you'll have error notification in both listeners.

### clickHouse.querying (statement, [options]).then (…)

Promise interface. Similar to the callback one.

### clickHouse.ping (function (err, response) {})

Sends an empty query and check if it "Ok.\n"

### clickHouse.pinging ().then (…)

Promise interface for `ping`

Notes
-----

## Memory size

You can read all the records into memory in single call like this:

```javascript

var ch = new ClickHouse ({host: host, port: port});
ch.query ("SELECT number FROM system.numbers LIMIT 10", {syncParser: true}, function (err, result) {
  // result will contain all the data you need
});

```

In this case whole JSON response from the server will be read into memory,
then parsed into memory hogging your CPU. Default parser will parse server response
line by line and emits events. This is slower, but much more memory and CPU efficient
for larger datasets.

## Promise interface

Promise interface have some restrictions. It is not recommended to use this interface
for `INSERT` and `SELECT` queries. For the `INSERT` you cannot bulk load data via stream,
`SELECT` will collect all the records in the memory. For simple usage where data size
is controlled it is ok.
