---
layout: page
title: "Example: package-manager.js"
permalink: /examples/package-manager
---

```js
// package-manager.js
const fs = System.files

exports.init = async () => {
  await fs.mkdir('/meta')
  await fs.mkdir('/packages')
}

exports.publish = async (name, version, data) => {
  // validate params
  assert(name && typeof name === 'string', 'Name must be a string')
  assert(version && typeof version === 'string', 'Version must be a string')
  assert(data && typeof data === 'string', 'Data must be a base64ed string of the package data')

  // setup paths
  const metaPath = `/meta/${name}.json`
  const dataPath = `/packages/${name}.${version}`

  // read metadata record
  var meta
  try { meta = JSON.parse(await fs.readFile(metaPath)) }
  } catch (e) {}

  // check ownership if it already exists
  if (meta) {
    assert(meta.owner === System.caller.id, 'This package name is already taken')
  }

  // write metadata
  await fs.writeFile(metaPath, JSON.stringify({
    owner: System.caller.id,
    name,
    version
  }))

  // write data
  await fs.writeFile(dataPath, data, 'base64')
}

function assert (cond, err) {
  if (!cond) throw new Error(err)
}
```

<br>

## Using package-manager.js

Start the server:

```bash
$ nodevms exec package-manager.js --dir ~/packages
Serving at localhost:5555
Serving directory /home/alice/packages

Files:    dat://17f29b83be7002479d8865dad3765dfaa9aaeb283289ec65e30992dc20e3dabd
Call log: dat://7081814137ea43fc32348e2259027e94e85c7b395e6f3218e5f5cb803cc9bbef
```

Then connect to the service to add packages:

```bash
$ nodevms repl localhost:5555 \
  --user alice \ # using debugmode authentication
  --exec "client.publish('foo.js', '1.0.0', $(cat foo.js | base64))"
```

You can view the current packages using the [Dat CLI](https://npm.im/dat) or [Beaker Browser](https://beakerbrowser.com/).

```bash
$ dat \
  dat://17f29b83be7002479d8865dad3765dfaa9aaeb283289ec65e30992dc20e3dabd/ \
  ./pkg
$ cd ./pkg/data
$ ls
foo.js.1.0.0
$ cat foo.js.1.0.0
console.log('Hello, world!')
```