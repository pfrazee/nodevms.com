---
layout: page
title: "Example: guest-list.js"
permalink: /examples/guest-list
---

```js
// guest-list.js
exports.init = async () => {
  await System.files.mkdir('/rsvps')
}

// method to tell the host if you're attending the party
exports.RSVP = async (name, isAttending, reason) => {
  // validate parameters
  if (!name || typeof name !== 'string') {
    throw new Error('Name is required')
  }
  if (typeof isAttending !== 'boolean') {
    throw new Error('Must specify whether you are attending')
  }

  // write the RSVP as a file in /rsvps
  const id = System.caller.id // the caller's public key (authenticated)
  const path = `/rsvps/${id}.json`
  const record = {id, name, isAttending, reason}
  await System.files.writeFile(path, JSON.stringify(record))
}
```

<br>

## Using guest-list.js

Start the server:

```bash
$ nodevms exec guest-list.js --dir ~/guests
Serving at localhost:5555
Serving directory /home/alice/guests

Files:    dat://17f29b83be7002479d8865dad3765dfaa9aaeb283289ec65e30992dc20e3dabd
Call log: dat://7081814137ea43fc32348e2259027e94e85c7b395e6f3218e5f5cb803cc9bbef
```

Users connect to the service to RSVP:

```bash
$ nodevms repl localhost:5555 \
  --user bob \ # using debugmode authentication
  --exec "client.RSVP('bob', false, 'Washing my hair, sorry!')"
```

You can view the current RSVPs using the [Dat CLI](https://npm.im/dat) or [Beaker Browser](https://beakerbrowser.com/).

```bash
$ dat \
  dat://17f29b83be7002479d8865dad3765dfaa9aaeb283289ec65e30992dc20e3dabd/ \
  ./scriptdata
$ cd ./scriptdata/rsvps
$ ls
bob.json
$ cat bob.json
{"id":"bob","name":"bob","isAttending":"false","reason":"Washing my hair, sorry!"}
```