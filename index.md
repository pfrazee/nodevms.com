---
layout: page
title: A cryptographically auditable VM service
---

NodeVMS is a server which provides external auditability of its state and behaviors using [secure ledgers](https://beakerbrowser.com/2017/06/19/cryptographically-secure-change-feeds.html).

```bash
# Run auditable services
$ nodevms exec ./backend-script.js

# Run commands on a remote service
$ nodevms repl localhost:5555

# Audit the state and history of a service
$ nodevms verify localhost:5555
```

**Alpha Software**. Not ready for use in production.

<br>

## How does it work?

VMS uses [Dat's](https://github.com/datproject/dat) secure ledger and files distribution to publish transactions and service state in a public, unforgeable format. Clients can then download, replay, audit, and compare the state of the service to ensure the declared code is being executed correctly.

The core of NodeVMS is a code-contract which has its execution verified by third parties. On creation, the contract is set permanently in the log (the "secure ledger"). Thereafter, all activity and state is written to the ledger, and can be monitored from any other computer.

As the service runs, the monitoring computers watch the ledger to make sure any state-change matches the contract and the remote requests. If the service deviates from the code-contract, the deviation will be detected by the monitors. (This is a similar design to [Certificate Tranparency](https://www.certificate-transparency.org/).)

NodeVMS is a proof-of-concept. It is comparable to decentralized smart-contract platforms such as Ethereum. The key difference is that NodeVMS uses only one hosting server. This makes NodeVMS much cheaper to run, as no Proof of Work is required. However, it does not decentralize operation of the contract.

NodeVMS would be useful for running name servers, encrypted key distribution, or any other kind of database that requires a high degree of outside auditability.

<br>

## Usecases

**Usecase 1: Auditing high-value datasets**

There are a class of services which require a high amount of trust in the execution of the service. One example is key distribution and identity. Another example is code distribution. Users need to trust that the code is executed correctly, and the correct data is then distributed.

For these cases, an auditable service gives confidence that the service isn’t lying. Outputs can be traced to their sources, and unexpected changes to state can be flagged publicly.

**Usecase 2: Execution on untrusted hardware**

Most daily computing on the Web is simple to execute. It requires basic trust that the host will stay available online and run the code correctly.

If we can guarantee availability and correctness through auditing, it becomes much easier to share computing power with strangers. We could share server CPUs the same way we share the WiFi at a coffee shop. 

<br>

## Overview

NodeVMS provisions service contracts. These contracts export an RPC interface which can be accessed by the network. Additionally, each contract has a networked filesystem which it can read and write to.

Like Blockchains, NodeVMS uses a secure ledger for auditability. Each VM host publishes a "call log" which includes:

 - the script executed by the VM,
 - the public key of the files archive, and
 - all requests and responses to the VM host.

Further, each request in the log includes:

 - a signature by the caller, 
 - the changes to the file's archive version,
 - and the pubkey of the call ledger.

For auditing, a client downloads the call ledger, then provisions the VM locally and replays the log. As it plays each call, it checks the return values and filesystem state. If there is a mismatch from the hosted VM, the auditor knows that either

 1. the VM host modified the state outside the log, or 
 2. the VM script is nondeterministic.

These audits are the responsibility of the clients, and can be periodically run as a background task for the VMs you provision.

<br>

## What's the goal?

VMS hosts would be integrated with a browser (such as [Beaker](https://beakerbrowser.com)) so that users can quickly self-deploy backend scripts, either on their personal device or on a public host. This will enable apps which ship with backend scripts, and can provision backend services in order to maintain shared datasets.

For instance, this "RSVP" backend script might be used on an event site to track guests:

```js
// RSVP.js
exports.init = async () => {
  await System.files.mkdir('/rsvps')
}
exports.RSVP = async (isAttending, reason) => {
  await System.files.writeFile('/rsvps/${System.caller.id}.json', JSON.stringify({
    who: System.caller.id,
    isAttending,
    reason
  }))
}
```

This script would be deployed on a VMS host and given a url such as `wss://nodevms.com/bob/my-party-rsvps`. An app would then connect to the backend to transact:

```js
// bobs-party-page.js
const NodeVMSClient = require('nodevms-client')
const VMS_URL = 'wss://nodevms.com/bob/my-event-rsvps'
async function onClickYes () {
  const client = new NodeVMSClient()
  await client.connect(VMS_URL)
  await client.RSVP(true)
}
async function onClickNo () {
  const client = new NodeVMSClient()
  await client.connect(VMS_URL)
  await client.RSVP(false, prompt('Why tho?'))
}
```

The current state of the RSVPs could then be read directly by accessing the backend's files archive.

```js
// bobs-party-page.js
async function getRSVPs () {
  const client = new NodeVMSClient()
  await client.connect(VMS_URL)
  const fs = new DatArchive(client.backendInfo.filesArchiveUrl)
  const rsvpFilenames = await fs.readdir('/rsvps')
  let rsvps = []
  for (let i = 0; i < rsvpFilenames.length; i++) {
    let filename = rsvpFilenames[i]
    rsvps.push(JSON.parse(await fs.readFile(`/rsvps/${filename}`)))
  }
  return rsvps
}
```