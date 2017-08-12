---
layout: page
title: Tutorial
permalink: /tutorial/
---

A "backend script" is a self-contained nodejs module.

```js
// counter.js
var i = 0
exports.increment = function () {
  i++
  return i
}
```

It exports methods which can be called over RPC.

To serve the backend script, we use the commandline. First, install it with

```bash
npm install -g nodevms
```

Then:

```bash
$ nodevms exec ./counter.js
Serving at localhost:5555
Serving directory /home/bob/counter

Files:    dat://17f29b83be7002479d8865dad3765dfaa9aaeb283289ec65e30992dc20e3dabd
Call log: dat://7081814137ea43fc32348e2259027e94e85c7b395e6f3218e5f5cb803cc9bbef
```

Clients can now connect and call to the backend!

Let's use the NodeVMS REPL to do so:

```bash
$ nodevms repl localhost:5555
Connecting...
Connected.
You can use 'client' object to access the backend.
> client.increment()
1
> client.increment()
2
> client.increment()
3
```

Great! We have a backend service that maintains a counter for us. We can increment the counter by calling its exported method `increment()`.

### Persisting state to the files dat-archive

There's only one problem: the state isn't being persisted anywhere. If you were to restart NodeVMS, the counter will reset to zero. That isn't very useful.

To fix that, we need to persist state to the backend's files archive.

```js
// persistent-counter.js
const fs = System.files
exports.increment = async function () {
  var i = await fs.readFile('/counter', 'json')
  i++
  await fs.writeFile('/counter', i)
  return i
}
```

Now, the counter state will persist after restarting the backend script.

### The files archive (`System.files`)

The files dat-archive provides a sandboxed folder for keeping state. Its interface can be found on the global `System` object as `System.files`.

You can share the backend's files dat-archive. In fact, that is the recommended way to have people read the state of the backend! Its URL is emitted at start:

```
Files:    dat://17f29b83be7002479d8865dad3765dfaa9aaeb283289ec65e30992dc20e3dabd
```

An example of how you might use the files dat-archive is, you might write a backend to maintain a photo album. The backend script would simply provide an API for writing the images:

```js
const fs = System.files
exports.addPhoto = async function (name, data, encoding) {
  const path = `/photos/${name}`
  var alreadyExists = await doesFileExist(path)
  if (alreadyExists) throw new Error('File already exists')
  await fs.writeFile(path, data, encoding)
}
async function doesFileExist (path) {
  try {
    await fs.stat(path)
    return true
  } catch (e) {
    return false
  }
}
```

This backend would ensure that each name for a photo can be taken once-and-only-once.

### The content of the files dat-archive

The state of the files dat-archive is saved on the FS of the NodeVMS server. Its location is also emitted at the start (the "Serving directory") and it can be configured via cli opts:

```
$ nodevms exec ./counter.js --dir ./my-counter-files
```

If you examine the directory, you will find the internal datastructures of the files dat-archive and the call log.

**NOTE: You should never change the content of the files!** Clients of your backend expect to be able to audit all changes made to the backend's state. They *will* detect an unlogged change and lose trust in your backend.

### Auditing the state of a backend

Each backend executed by NodeVMS publishes a call log using [Dat](https://beakerbrowser.com/docs/inside-beaker/dat-files-protocol.html). This call log can be replayed using NodeVMS to verify the state of the files dat-archive.

```bash
$ nodevms verify localhost:5555
›Connecting to ws://localhost:5555/...
›Connected.
›Downloading call log...
›Downloading files archive...
›Replaying 4 calls...
›Comparing outputs...
✔Call log verified.
✔Output files verified.
```

Optionally, you can include the urls of the expected files archive and dat log:

```bash
$ nodevms verify localhost:5555 \
  --files dat://17f29b83be7002479d8865dad3765dfaa9aaeb283289ec65e30992dc20e3dabd \
  --log dat://7081814137ea43fc32348e2259027e94e85c7b395e6f3218e5f5cb803cc9bbef
```

Your NodeVMS client will download the call log and the current files archive, then replay the history to confirm the output state. If it does not match, NodeVMS will alert you to the disparity.


### Users & authentication

The backend is provided information about the calling user, in order to make permissions decisions. The user's id is located on the `System` object, as `System.caller.id`. Here's a simple example usage of permissions:

```js
// secure-counter.js
var ownerId
exports.claimOwnership = () => {
  if (ownerId) throw new Error('I already have an owner!')
  ownerId = System.caller.id
}
var counter = 0
exports.increment = () => {
  if (System.caller.id !== ownerId) throw new Error('You are not my owner!')
  return counter++
}
```

This backend provides a counter which only the owner can increment. (The owner is established as the first user to connect and call `claimOwnership()`.) The call log will note the caller ID for every call, along with the caller's signature on the call data.

When debugmode is on, you can set the caller id to anything using the Basic Auth header when connecting to the server's websocket. For example:

```
$ nodevms exec ./secure-counter.js --debug

# in another term, the repl call:
$ nodevms repl localhost:5555 --user bob
```

**NOTE: The current version of NodeVMS does not have production authentication implemented.** Only the debugmode authentication is available.

### Calling out of the backend and accessing "Oracles"

The current version of NodeVMS does not let the backend-script access other processes, the network, or the FS (other than the files dat-archive). This is for two reasons:

 1. It's safer to run untrusted backends if they are sandboxed, and
 2. It encourages deterministic and auditable backends.

This means that, at this time, you cannot contact "Oracles."

**What is an Oracle?** An Oracle is a source of information that cannot be audited, usually because the source of its information is not auditably modeled in an backend. Put another way, it is a black box which a backend consults. Examples of Oracles include: sensors (eg a thermometer), random number generators, the wall clock, and a stock-price service. Any time an Oracle is used, it has to be trusted by the users, and it has to be modeled specially to deal with nondeterminism.

**About nondeterminism.** Backend scripts are designed to be deterministic. Their output state is a function of their call log: if you replay the call log against the backend script, you should get the same files archive. However, Oracles are nondeterministic- they introduce information which is not provided by the call log. There is currently no way to model Oracles or indeterminism in NodeVMS. If your backend's effects are not fully deterministic, then there is a good chance that verification will fail.

This is an example of a simple non-deterministic backend:

```js
exports.rand = async function () {
  return Math.random() // ignore the fact that we could seed this random
}
```

The return value is not replayable in this backend, and so an audit would fail.

In the future, there will be a way to record non-determinism -- essentially by wrapping areas of code and storing the output in the call log. It will look something like this:

```js
exports.rand = async function () {
  return await System.oracle(() => Math.random())
}
```

The `System.oracle()` wrapper will cache the return value so that replays of the log can use the same values, and not call the internal logic.

### The `exports.init()` method

If the script exports a `.init` method, it will be called on initial setup. This is to enable preparation on the files archive (ie to create folders and initial state). After the first call, it will never be called again and is not exposed as an RPC endpoint.

### The script-wide lock

Because it's important that a NodeVMS backend has deterministic results, NodeVMS only executes one RPC call at a time -- all other calls are queued until the active call returns. This is called a "script-wide lock." The script-wide lock is less efficient, but it improves the replayability of the backend.

### JS client

You can programmatically connect to a NodeVMS backend using `nodevms-client`:

```js
var RPCClient = require('nodevms-client')
var client = new RPCClient()
await client.connect('localhost:5555')
console.log(await client.increment())
```

The rpc object will have all of the methods exported by the backend. Each method returns a promise, and can take any number of arguments.

The `connect()` function takes a set of opts:

```js
RPC.connect(backendURL, {
  user: 'bob'  // who should we connect as? (default null)
})
```

### Further reading

 - [LibVMS documentation](https://github.com/pfrazee/libvms) (API docs for NodeVMS)
 - [About the Dat files protocol](https://beakerbrowser.com/docs/inside-beaker/dat-files-protocol.html)
 - [About secure ledgers](https://beakerbrowser.com/2017/06/19/cryptographically-secure-change-feeds.html)