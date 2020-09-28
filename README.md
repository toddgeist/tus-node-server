# tus-node-server
[![npm version](https://badge.fury.io/js/tus-node-server.svg)](https://badge.fury.io/js/tus-node-server)
[![Build Status](https://travis-ci.org/tus/tus-node-server.svg?branch=master)](https://travis-ci.org/tus/tus-node-server)
[![Coverage Status](https://coveralls.io/repos/github/tus/tus-node-server/badge.svg?branch=master)](https://coveralls.io/github/tus/tus-node-server?branch=master)
[![Dependency Status](https://david-dm.org/tus/tus-node-server.svg)](https://david-dm.org/tus/tus-node-server#info=dependencies)
[![devDependencies Status](https://david-dm.org/tus/tus-node-server/dev-status.svg)](https://david-dm.org/tus/tus-node-server?type=dev)

tus is a new open protocol for resumable uploads built on HTTP. This is the [tus protocol 1.0.0](http://tus.io/protocols/resumable-upload.html) node.js server implementation.

> :warning: **Attention:** We currently lack the resources to properly maintain tus-node-server. This has the unfortunate consequence that this project is in rather bad condition (out-dated dependencies, no tests for the S3 storage, no resumable uploads for the GCS storage etc). If you want to help us with tus-node-server, we are more than happy to assist you and welcome new contributors. In the meantime, we can recommend [tusd](https://github.com/tus/tusd) as a reliable and production-tested tus server. Of course, you can use tus-node-server if it serves your purpose.

## Installation

```bash
$ npm install tus-node-server
```

## Flexible Data Stores

- **Local File Storage**
    ```js
    server.datastore = new tus.FileStore({
        path: '/files'
    });
    ```

- **Google Cloud Storage**
    ```js

    server.datastore = new tus.GCSDataStore({
        path: '/files',
        projectId: 'project-id',
        keyFilename: 'path/to/your/keyfile.json',
        bucket: 'bucket-name',
    });
    ```

- **Amazon S3**
    ```js

    server.datastore = new tus.S3Store({
        path: '/files',
        bucket: 'bucket-name',
        accessKeyId: 'access-key-id',
        secretAccessKey: 'secret-access-key',
        region: 'eu-west-1',
        partSize: 8 * 1024 * 1024, // each uploaded part will have ~8MB,
        tmpDirPrefix: 'tus-s3-store',
    });
    ```

## Quick Start

#### Use the [tus-node-deploy](https://hub.docker.com/r/bhstahl/tus-node-deploy/) Docker image

```sh
$ docker run -p 1080:8080 -d bhstahl/tus-node-deploy
```

#### Build a standalone server yourself
```js
const tus = require('tus-node-server');

const server = new tus.Server();
server.datastore = new tus.FileStore({
    path: '/files'
});

const host = '127.0.0.1';
const port = 1080;
server.listen({ host, port }, () => {
    console.log(`[${new Date().toLocaleTimeString()}] tus server listening at http://${host}:${port}`);
});
```

#### Use tus-node-server as [Express Middleware](http://expressjs.com/en/guide/using-middleware.html)

```js
const tus = require('tus-node-server');
const server = new tus.Server();
server.datastore = new tus.FileStore({
    path: '/files'
});

const express = require('express');
const app = express();
const uploadApp = express();
uploadApp.all('*', server.handle.bind(server));
app.use('/uploads', uploadApp);

const host = '127.0.0.1';
const port = 1080;
app.listen(port, host);
```

#### Use tus-node-server with [Koa](https://github.com/koajs/koa) or plain Node server

```js
const http = require('http');
const url = require('url');
const Koa = require('koa')
const tus = require('tus-node-server');
const tusServer = new tus.Server();

const app = new Koa();
const appCallback = app.callback();
const port = 1080;

tusServer.datastore = new tus.FileStore({
    path: '/files',
});

const server = http.createServer((req, res) => {
    const urlPath = url.parse(req.url).pathname;

    // handle any requests with the `/files/*` pattern
    if (/^\/files\/.+/.test(urlPath.toLowerCase())) {
        return tusServer.handle(req, res);
    }

    appCallback(req, res);
});

server.listen(port)
```

## Features
#### Events:

Execute code when lifecycle events happen by adding event handlers to your server.

```js
const Server = require('tus-node-server').Server;
const EVENTS = require('tus-node-server').EVENTS;

const server = new Server();
server.on(EVENTS.EVENT_UPLOAD_COMPLETE, (event) => {
    console.log(`Upload complete for file ${event.file.id}`);
});
```

- `EVENT_FILE_CREATED`: Fired when a `POST` request successfully creates a new file

    _Example payload:_
    ```
    {
        file: {
            id: '7b26bf4d22cf7198d3b3706bf0379794',
            upload_length: '41767441',
            upload_metadata: 'filename NDFfbWIubXA0'
         }
    }
    ```

- `EVENT_ENDPOINT_CREATED`: Fired when a `POST` request successfully creates a new upload endpoint

    _Example payload:_
    ```
    {
        url: 'http://localhost:1080/files/7b26bf4d22cf7198d3b3706bf0379794'
    }
    ```

- `EVENT_UPLOAD_COMPLETE`: Fired when a `PATCH` request finishes writing the file

    _Example payload:_
    ```
    {
        file: {
            id: '7b26bf4d22cf7198d3b3706bf0379794',
            upload_length: '41767441',
            upload_metadata: 'filename NDFfbWIubXA0'
        }
    }
    ```

#### Custom `GET` handlers:
Add custom `GET` handlers to suit your needs, similar to [Express routing](https://expressjs.com/en/guide/routing.html).
```js
const server = new Server();
server.get('/uploads', (req, res) => {
    // Read from your DataStore
    fs.readdir(server.datastore.path, (err, files) => {
        // Format the JSON response and send it
    }
});
```

#### Custom file names:
```js
const fileNameFromUrl = (req) => {
    return req.url.replace(/\//g, '-');
}

server.datastore = new tus.FileStore({
    path: '/files',
    namingFunction: fileNameFromUrl
});
```

## Development

Start the demo server using Local File Storage
```bash
$ npm run demo
```

Or start up the demo server using Google Cloud Storage
```bash
$ npm run gcs_demo
```

Then navigate to the demo ([localhost:1080](http://localhost:1080)) which uses [`tus-js-client`](https://github.com/tus/tus-js-client)
