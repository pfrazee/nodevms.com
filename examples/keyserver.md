---
layout: page
title: "Example: keyserver.js"
permalink: /examples/keyserver
---

```js
// keyserver.js
const fs = System.files

exports.init = async () => {
  await fs.mkdir('/keys')
}

// key management
// =

exports.setKey = async (name, pubkey) => {
  // validate parameters
  assert(name && typeof name === 'string', 'Name param must be a string')
  assert(pubkey && typeof pubkey === 'string', 'Pubkey param must be a string')

  // validate perms
  var admins = await getAdmins()
  assertCallerIsAdmin(admins)

  // write the key binding as a file
  await fs.writeFile(`/keys/${name}`, pubkey)
}

exports.delKey = async (name) => {
  // validate parameters
  assert(name && typeof name === 'string', 'Name param must be a string')

  // validate perms
  var admins = await getAdmins()
  assertCallerIsAdmin(admins)

  // delete the file
  await fs.unlink(`/keys/${name}`)
}

// admin controls
// =

// method to set the admins of this key server
// - expects [{id: String, name: String}, ...]
exports.setAdmins = async (newAdmins) => {
  var existAdmins = await getAdmins()
  assertValidAdminArray(existAdmins)
  assertCallerIsAdmin(existAdmins)
  await fs.writeFile('/admins', JSON.stringify(newAdmins))
}

// helper to fetch the current admins
async function getAdmins () {
  try { return JSON.parse(await fs.readFile('/admins')) }
  catch (e) { return [] }
}

// helper to validate the admins array
function assertValidAdminArray (admins) {
  assert(Array.isArray(admins), 'Admins must be an array')
  admins.forEach(admin => {
    assert(typeof admin === 'object', 'Admin items must be a {id:, name:}')
    assert(admin.id && typeof admin.id === 'string', 'Admin items must be a {id:, name:}')
    assert(admin.name && typeof admin.name === 'string', 'Admin items must be a {id:, name:}')
  })
}

// throw if the given caller is not an admin
function assertCallerIsAdmin (admins) {
  if (!admins || admins.length === 0) {
    return true // no admins yet, allowed
  }
  assert(admins.find(a => a.id === System.caller.id), 'Not allowed')
}

function assert (cond, err) {
  if (!cond) throw new Error(err)
}
```

<br>

## Using keyserver.js

Start the server:

```bash
$ nodevms exec keyserver.js --dir ~/keyserver
Serving at localhost:5555
Serving directory /home/alice/keyserver

Files:    dat://17f29b83be7002479d8865dad3765dfaa9aaeb283289ec65e30992dc20e3dabd
Call log: dat://7081814137ea43fc32348e2259027e94e85c7b395e6f3218e5f5cb803cc9bbef
```

Connect to the service to setup the admin:

```bash
$ nodevms repl localhost:5555 \
  --user alice \ # using debugmode authentication
  --exec "client.setAdmins([{id: 'alice', name: 'Alice'}])"
```

Now set keys as needed:

```bash
$ nodevms repl localhost:5555 \
  --user alice \ # using debugmode authentication
  --exec "client.setKey('bob', 'c7b395e6f3218e5f5cb803cc9bbef7081814137ea43fc32348e2259027e94e85')"
```

You can view the current keys using the [Dat CLI](https://npm.im/dat) or [Beaker Browser](https://beakerbrowser.com/).

```bash
$ dat \
  dat://17f29b83be7002479d8865dad3765dfaa9aaeb283289ec65e30992dc20e3dabd/ \
  ./keydata
$ cd ./keydata/keys
$ ls
bob
$ cat bob
c7b395e6f3218e5f5cb803cc9bbef7081814137ea43fc32348e2259027e94e85
```