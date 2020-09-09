# Multer Minio Storage Engine

Streaming multer storage engine for minio.

This project is mostly an integration piece for existing code samples from Multer's [storage engine documentation](https://github.com/expressjs/multer/blob/master/StorageEngine.md) with [MinIO Client SDK for Javascript](https://github.com/minio/minio-js) as the substitution piece for file system. Existing solutions I found required buffering the multipart uploads into the actual filesystem which is difficult to scale.

## Installation

```sh
yarn add multer-minio-storage-engine
```

## Usage

```javascript
const Minio = require('minio');
const express = require('express');
const multer = require('multer');
const multerMinio = require('multer-minio-storage-engine');

const app = express();

const minioClient = new Minio.Client({
  /* ... */
});

const upload = multer({
  storage: multerMinio({
    minio: minioClient,
    bucketName: 'some-bucket',
    metaData: function (req, file, cb) {
      cb(null, {mimetype: file.mimetype});
    },
    objectName: function (req, file, cb) {
      cb(null, Date.now().toString());
    },
    preprocess: {
      '.suffix1': stream => stream,
      '.suffix2': stream => stream,
      '.suffix3': stream => stream,
    }
  }),
});

app.post('/upload', upload.array('photos', 3), function (req, res, next) {
  res.send('Successfully uploaded ' + req.files.length + ' files!');
});
```

### File information

Each file contains the following information exposed by `multer-minio-storage-engine`:

| Key           | Description                            | Note        |
| ------------- | -------------------------------------- | ----------- |
| `size`        | Size of the file in bytes              |
| `bucketName`  | The bucket used to store the file      | `S3Storage` |
| `objectName`  | The name of the object                 | `S3Storage` |
| `contentType` | The `mimetype` used to upload the file | `S3Storage` |
| `metaData`    | The `metaData` object to be sent to S3 | `S3Storage` |
| `etag`        | The `etag`of the uploaded file in S3   | `S3Storage` |

## Setting MetaData

The `metaData` option is a callback that accepts the request and file, and returns a metaData object to be saved to S3.

Here is an example that stores all fields in the request body as metaData, and uses an `id` param as the objectName:

```javascript
const opts = {
  minio: minioClient,
  bucketName: 'some-bucket',
  metaData: function (req, file, cb) {
    cb(null, Object.assign({}, req.body));
  },
  objectName: function (req, file, cb) {
    cb(null, req.params.id + '.jpg');
  },
};
```

## Preprocessing streams before upload

The `preprocess` option is an object that includes key-value pairs for transforming the input stream into multiple individual file objects. The key is a suffix that will be appended to the object's `objectName` and the value is a function that takes the input stream and should return a stream.

For example, you can use this feature in conjunction with [gm](https://www.npmjs.com/package/gm) to produce multiple different sizes of the uploaded image:

```javascript
const gm = require('gm').subClass({imageMagick: true})
const opts = {
  minio: minioClient,
  bucketName: 'some-bucket',
  metaData: function (req, file, cb) {
    cb(null, Object.assign({}, req.body));
  },
  objectName: function (req, file, cb) {
    cb(null, req.params.id + '.jpg');
  },
  preprocess: {
    "(100x100)": stream => gm(stream).resize(100, 100).stream(), // Creates an object with (100x100) at the end
    "(200x200)": stream => gm(stream).resize(200, 200).stream(), // for example if the object name is "upload.jpg)
    "(300x300)": stream => gm(stream).resize(300, 300).stream(), // we will have "upload.jpg(100x100)"
  }
};
```

<!--
## Setting Custom Content-Type

The optional `contentType` option can be used to set Content/mime type of the file. By default the content type is set to `application/octet-stream`. If you want multer-minio-storage-engine to automatically find the content-type of the file, use the `multerMinio.AUTO_CONTENT_TYPE` constant. Here is an example that will detect the content type of the file being uploaded.

```javascript
const upload = multer({
  storage: multerMinio({
    minio: minioClient,
    bucketName: 'some-bucket',
    contentType: multerMinio.AUTO_CONTENT_TYPE,
    objectName: function (req, file, cb) {
      cb(null, Date.now().toString());
    },
  }),
});
```

You may also use a function as the `contentType`, which should be of the form `function(req, file, cb)`. -->

<!--
## Testing

The tests mock all access to Minio and can be run completely offline.

```sh
npm test
``` -->
