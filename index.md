---
layout: page
title: A smart contract server built with Dat
---

NodeVMS is a smart-contract server which provides external auditability of its state and behaviors using Dat's [secure ledgers](https://beakerbrowser.com/2017/06/19/cryptographically-secure-change-feeds.html).

```js
// contract.js (a simple guest list example)
exports.init = async () => {
  await System.files.mkdir('/rsvps')
}

// method to tell the host if you're attending the party
exports.RSVP = async (name, isAttending, reason) => {
  // -- validate parameters here --

  // write the RSVP as a file in /rsvps
  const id = System.caller.id
  const path = `/rsvps/${id}.json`
  const record = {id, name, isAttending, reason}
  await System.files.writeFile(path, JSON.stringify(record))
}
```

On the command line:

```bash
# Run auditable services
$ nodevms exec ./contract.js

# Run commands on a remote service
$ nodevms repl localhost:5555 \
  --user bob \ # using debugmode authentication
  --exec "client.RSVP('bob', false, 'Washing my hair, sorry!')"

# Audit the state and history of a service
$ nodevms verify localhost:5555
```

<br>

## Demo video

<iframe width="560" height="315" src="https://www.youtube.com/embed/Qp38HOYcqW4" frameborder="0" allowfullscreen></iframe>

*Note: each request creates a script-wide lock until it completes, and so the implementation of persisted-counter.js is correct.*

<br>

## How does it work?

VMS uses [Dat's](https://datproject.org) secure ledger and files-archive to publish transactions and service state in a public, unforgeable format. Clients can then download, replay, audit, and compare the state of the service to ensure the declared code is being executed correctly.

The core of NodeVMS is a code-contract which has its execution verified by third parties. On creation, the contract is set permanently in the log (the "secure ledger"). Thereafter, all activity and state is written to the ledger, and can be monitored from any other computer.

> See also: ["What is a secure ledger?"](https://beakerbrowser.com/2017/06/19/cryptographically-secure-change-feeds.html)

As the service runs, the monitoring computers watch the ledger to make sure any state-change matches the contract and the remote requests. If the service deviates from the code-contract, the deviation will be detected by the monitors. (This is a similar design to [Certificate Transparency](https://www.certificate-transparency.org/).)

NodeVMS is a proof-of-concept. It is comparable to decentralized smart-contract platforms such as [Ethereum](https://ethereum.org/). The key difference is that NodeVMS uses only one hosting server. This makes NodeVMS much cheaper to run, as no Proof of Work is required. However, it does not decentralize operation of the contract.

NodeVMS would be useful for running name servers, encrypted key distribution, or any other kind of database that requires a high degree of outside auditability.