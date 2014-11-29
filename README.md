# numtel:mysql
Reactive MySQL for Meteor

Wrapper of the [MySQL NPM module](https://github.com/felixge/node-mysql) with a few added methods.

* [Quick tutorial on using this package](getting_started.md)
* [Leaderboard example modified to use MySQL](https://github.com/numtel/meteor-mysql-leaderboard)
* [Explanation of the implementation design process](context.md)

## Server Implements

### `mysql`

The `mysql` object is exposed to the server side of your application when this package is installed.

See the [MySQL NPM module documentation on GitHub](https://github.com/felixge/node-mysql) for very detailed instructions on this object.

If you are currently using the `pcel:mysql` package, the interface will be exactly the same except for the following methods which are added to each connection.

### `connection.queryEx(query)`

Perform any query synchronously and return the result. The single argument may contain a string or a function. A function may be passed that accepts two arguments. See example:

```javascript
var result = db.queryEx(function(esc, escId){
  return 'update ' + escId(table) +
         ' set `score`=`score` + ' + esc(amount) +
         ' where `id`=' + esc(id);
});
```
* The first argument, `esc` is a function that escapes values in the query.
* The second argument, `escId` is a function that escapes identifiers in the query.

### `connection.initUpdateTable(tableName)`

Specify a table (as string) to use for storing the keys used for notifying updates to queries. The table will be created if it does not exist. To install for the first time, specify a table name that does not currently exist.

This method must be called before any calls to `connection.select()`.

### `connection.select(subscription, options)`

Bind a select statement to a subscription object (e.g. context of a `Meteor.publish()` function). The first two options are required:

Option | Type | Description
------|-------|--------------
`query`|`string` or `function` | Query to perform (queryEx syntax)
`triggers`|`array`| Description of triggers to refresh query
`pollInterval` | `number` | Poll delay duration in milliseconds (Optional)

Every live-select utilizes the same poll timer. Passing a `pollInterval` will update the global poll delay. By default, the poll is intialized at 1000 ms. An interval too short may introduce erratic updates.

Each trigger object may contain the following properties:

Name | Type | Description
-----|-------| --------
`table` | `string` | Name of table to hook trigger *(Required)*
`condition` | `string` or `function` | Specify conditional terms to trigger *(Optional)* Access new row on insert or old row on update/delete using `$ROW` symbol. Example: `$ROW.name = "dude" or $ROW.score > 200`

## Client/Server Implements

### `MysqlSubscription([connection,] name, [args...])`

Constructor for subscribing to a published select statement. No extra call to `Meteor.subscribe()` is required. Specify the name of the subscription along with any arguments.

The first argument, `connection`, is optional. If connecting to a different Meteor server, pass the DDP connection object in this first argument. If not specified, the first argument becomes the name of the subscription (string) and the default Meteor server connection will be used.

The prototype inherits from `Array`, extended with the following extra methods and properties:

#### `on(eventName, handler)`

Bind a handler function to one of the available events on this subscription:

* `update` Called on all changes to the data, before the array is updated. Handler accepts 2 arguments: `index` (Index in subscription array) and `msg` (DDP message received).
* `added` Called when a row is inserted to the data, after array is updated. Handler accepts 2 arguments: `index` and `newRow`.
* `changed` Called when a row is changed, after array is updated. Handler accepts 3 arguments: `index`, `oldRow`, and `newRow`.
* `removed` Called when a row is removed from the data, after array is updated. Handler accepts 2 arguments: `index` and `oldRow`.

If an event handler returns `false` exactly, it will halt further events of the same type.

#### `removeHandler(eventName, handler)`

Remove a handler function from an event queue. Pass the function itself as the `handler` argument.

#### `dispatchEvent(eventName)`

Dispatch the handlers for a given event.

#### `dep`

`Tracker.Dependency` object

* Call `myLiveSelect.dep.depend()` in a template helper to ensure reactive updating.
* Call `myLiveSelect.dep.changed()` to signal new data in the array. This is automatically called when the query updates.

#### `subscription`

Reference to Meteor subscription object with `ready()` and `stop()` methods.

## License

MIT
