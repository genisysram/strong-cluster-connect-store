# Connect Session Store for Cluster

[![Build Status](https://travis-ci.org/strongloop/strong-cluster-connect-store.png?branch=master)](https://travis-ci.org/strongloop/strong-cluster-connect-store)
[![NPM version](https://badge.fury.io/js/strong-cluster-connect-store.png)](http://badge.fury.io/js/strong-cluster-connect-store)

## Overview

Strong-cluster-connect-store is an implementation of connect session store
using node's native cluster messaging. It provides an easy way for using
sessions in connect/express based applications running in a node cluster.

Features:

- Supports both Connect and Express.
- No dependencies on external services.
- Module is shipped without connect, it will use *your* version of connect
  or express.
- Covered by unit-tests.

See also [API documentation](http://apidocs.strongloop.com/strong-cluster-connect-store/).

## Installation

```sh
$ npm install strong-cluster-connect-store
```

## Use

### Configuration - connect

```js
var connect = require('connect');
var ClusterStore = require('strong-cluster-connect-store')(connect);

var app = connect();
app
  .use(connect.cookieParser())
  .use(connect.session({ store: new ClusterStore(), secret: 'keyboard cat' }));
```

### Configuration - express

```javascript
var express = require('express');
var ClusterStore = require('strong-cluster-connect-store')(express);

var app = express();
app
  .use(express.cookieParser())
  .use(express.session({ store: new ClusterStore(), secret: 'keyboard cat' }));
```

### Setting up the master process

The store requires that a shared-state server is running in the master process.
The server is initialized automatically when you require() this module
from the master. In the case that your master and workers have separate source
files, you must explicitly require this module in your master source file.
Optionally, you can call `setup()` to make it more obvious why you are loading
a module that is not used anywhere else.

```javascript
// master.js

var cluster = require('cluster');
// etc.

require('strong-cluster-connect-store').setup();

// configure your cluster
// fork the workers
// etc.
}
```

### Using Strong Cluster Connect Store

The following example assumes you have set up Strong Cluster Connect Store for Express and have run the steps above.

```javascript
'use strict';
var express = require('express');
var cluster = require('cluster');
var numCPUs = require('os').cpus().length;
var ClusterStore = require('strong-cluster-connect-store')(express.session);

if (cluster.isMaster) {
  // The cluster master executes this code

  ClusterStore.setup();

  // Create a worker for each CPU
  for (var i=0; i<numCPUs; i++) {
    cluster.fork();
  }

  cluster.on('online', function(worker) {
    console.log('Worker ' + worker.id + ' is online.');
  });

  cluster.on('exit', function(worker, code, signal) {
    console.log('worker ' + worker.id + ' died with signal',signal);
  });
} else {
  // The cluster workers execute this code

  var app = express();
  app.use(express.cookieParser());

  app.use(express.session(
    { store: new ClusterStore(), secret: 'super-cool' }
  ));

  app.get('/hello', function(req, res) {
    var msg;
    if (req.session.visited)
      msg = {msg: 'Hello again from worker '+cluster.worker.id};
    else
      msg = {msg: 'Hello from worker '+cluster.worker.id};

    req.session.visited = '1';
    res.json(200, msg);
  });
  app.listen(8080);
}
```
