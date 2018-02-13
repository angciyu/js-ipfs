# Tutorial - Understanding Circuit Relay

> Welcome! This tutorial will help you understand circuit relay, where it fits in the stack and how to use it.

## Application diagram

The goal of this tutorial is to explain what circuit relaying is, where it fits in the stack and how to use it.

```
Create a diagram
```

### So what is a `circuit-relay` and what do we need it for?

In p2p networks there are many cases where two nodes can't talk to each other directly. That may happen because of network topology, i.e. NATs, or execution environments - for example browser nodes can't connect to each other directly because they lack any sort of socket functionality and relaying on specialized rendezvous nodes introduces an undesirable centralization point to the network. A `circuit-relay` therefore is there to solve this problem. It is a node that allows two other nodes that can't otherwise talk to each other, use a third node, a relay to do so.

#### A word on circuit relay addresses
--- EXPAND ---

## Step-by-step instructions

Here's what we are going to be doing, today:

- 1. Spin up tree nodes - `js-ipfs` or `go-ipfs` and two browser nodes running the `examples/browser-webpack` example
- 2. Configure and start the relay nodes
- 3. Connect the two browser nodes to the `js-ipfs` circuit relay
- 4. Dial the two browser nodes using a `/p2p-circuit` address
- 5. Finally, adding a file in one of the browsers and retrieving it in another

Let's go.

### 1. Set up

You'll need to have an implementation of IPFS running on your machine. Currently, this means either go-ipfs or js-ipfs.

Installing go-ipfs can be done by installing the binary [here](https://ipfs.io/ipns/dist.ipfs.io/#go-ipfs). Alternatively, you could follow the instructions in the README at [ipfs/go-ipfs](https://github.com/ipfs/go-ipfs).

Installing js-ipfs requires you to have node and [npm](https://www.npmjs.com). Then, you simply run:

```sh
> npm install --global ipfs
...
> jsipfs --help
Commands:
...
```

This will alias `jsipfs` on your machine; this is to avoid issues with `go-ipfs` being called `ipfs`.

At this point, you have either js-ipfs or go-ipfs running. Now, initialize it:

```sh
> ipfs init
# or
> jsipfs init
```

This will set up your IPFS repo in your home directory.

### 2. Altering the `examples/exchange-files-in-browser` example

This tutorial takes advantage of the wonderful `exchange-files-in-browser` example to demonstrate the `circuit-relay` functionality, but we'll need to make one small change to it in order to make it suitable to our purposes. The current `exchange-files-in-browser` example uses the `websocket-star` transport in order to enable discovery and as a transport protocol, we don't want to use that, in fact, `circuit-relay` is here specifically to allow us to depart from the centralized nature of the `*-star` transports. 


#### Removing the `websockets-star` transport

Lets go ahead and open up the `examples/exchange-files-in-browser/public/app.js` and look for the `'/dns4/ws-star.discovery.libp2p.io/tcp/443/wss/p2p-websocket-star'` line, it should be inside the `Swarm` array:

```js
  Swarm: [
    // '/dns4/wrtc-star.discovery.libp2p.io/tcp/443/wss/p2p-webrtc-star'
    '/dns4/ws-star.discovery.libp2p.io/tcp/443/wss/p2p-websocket-star'
  ]
```

We can either comment it out, or remove it altogether - the idea is that we don't announce that address at all.


#### Removing `Bootstrap` nodes and enabling `circuit-relay`

We need to do one more change to the `examples/exchange-files-in-browser` in order to make things the way we want them. Namely, we want to remove the default bootstrap nodes, as well as enable `circuit-relay` in the browser example. Let's do it! 

We'll need to add the following entries to the `config` object right where we just commented out the `'/dns4/ws-star.discovery.libp2p.io/tcp/443/wss/p2p-websocket-star'` entry:

```js
Bootstrap: [],
EXPERIMENTAL: {
  relay: {
    enabled: true
  }
}
```

So that the final result looks like this:

```js
  config: {
    Addresses: {
      Swarm: [
        // '/dns4/wrtc-star.discovery.libp2p.io/tcp/443/wss/p2p-webrtc-star'
        // '/dns4/ws-star.discovery.libp2p.io/tcp/443/wss/p2p-websocket-star'
      ]
    },
    Bootstrap: [],
    EXPERIMENTAL: {
      relay: {
        enabled: true
      }
    }
  }
```

### 3. Configuring and starting a relay node

We can either use a `go-ipfs` or a `js-ipfs` node as a relay, we'll demonstrate how to set them up in this tutorial and we encourage you to try them both out. That said, either js or go, should do the trick for the purpose of this tutorial!

#### Setting up a `go-ipfs` node

In order to enable the relay functionality in `go-ipfs` we need to edit it's configuration file, located under `~/.ipfs/config`:

```js
  "Swarm": {
    "AddrFilters": null,
    "ConnMgr": {
      "GracePeriod": "20s",
      "HighWater": 900,
      "LowWater": 600,
      "Type": "basic"
    },
    "DisableBandwidthMetrics": false,
    "DisableNatPortMap": false,
    "DisableRelay": false,
    "EnableRelayHop": true
  }
```

The two options we're looking for are `DisableRelay` and `EnableRelayHop`. We want the former (`DisableRelay`) set to `false` and the later (`EnableRelayHop`) to `true`. That should set our go node as a relay - YAY!

#### Setting up a `js-ipfs` node

In order to do the same in `js-ipfs` we need to do something similar, but the config options are slightly different right now, that should change once this feature is not marked as experimental, but for now we have to deal with two different set of config options. Lets do this! Just as we did with `go-ipfs`, go ahead and edit `js-ipfs` config file located under `~/.jsipfs/config`, under the `EXPERIMENTAL` section. Lets add the following config:

(Note that the "EXPERIMENTAL" section might not be present in the config file, in that case, just go ahead and add it)

```js
  "EXPERIMENTAL": {
    "relay": {
      "enabled": true,
      "hop": {
        "enabled": true
      }
    }
  }
```

Again, that will make our `js-ipfs` node a relay node! Exciting!

### 4. Starting up the nodes

Now that we have everything configured, lets get those nodes started! 

#### Starting the relay node

We can start the relay nodes by either doing `ipfs daemon` or `jsipfs daemon`, that should start our previously installed go or js nodes. We should see an output similar to the bellow one:

```js
Initializing daemon...
Swarm listening on /p2p-circuit/ipfs/QmQR1DQNhDxsrk78ads6ccb8nDUG77c5tGK95HiXwQKUuV
Swarm listening on /p2p-circuit/ip4/0.0.0.0/tcp/4002/ipfs/QmQR1DQNhDxsrk78ads6ccb8nDUG77c5tGK95HiXwQKUuV
Swarm listening on /p2p-circuit/ip4/127.0.0.1/tcp/4003/ws/ipfs/QmQR1DQNhDxsrk78ads6ccb8nDUG77c5tGK95HiXwQKUuV
Swarm listening on /ip4/127.0.0.1/tcp/4003/ws/ipfs/QmQR1DQNhDxsrk78ads6ccb8nDUG77c5tGK95HiXwQKUuV
Swarm listening on /ip4/127.0.0.1/tcp/4002/ipfs/QmQR1DQNhDxsrk78ads6ccb8nDUG77c5tGK95HiXwQKUuV
Swarm listening on /ip4/192.168.1.132/tcp/4002/ipfs/QmQR1DQNhDxsrk78ads6ccb8nDUG77c5tGK95HiXwQKUuV
API is listening on: /ip4/127.0.0.1/tcp/5002
Gateway (readonly) is listening on: /ip4/127.0.0.1/tcp/9090
Daemon is ready
```

Look out for an address similar to `/ip4/127.0.0.1/tcp/4003/ws/ipfs/...` note it down somewhere, and lets move on to the next step.

#### Starting the browser nodes

Lets switch to the `examples/exchange-files-in-browser` that we altered in the previous steps, we'll need to run the usual `npm install` there to set the example up, once that's done, lets run `npm run start`, we should get an output similar to:

```js
Starting up http-server, serving public
Available on:
  http://127.0.0.1:12345
  http://192.168.1.132:12345
Hit CTRL-C to stop the server
```

Grab one of those addresses and paste it in your favorite browser (preferably the latest version of Firefox or Chrome). Now open another tab and past it again. You should be greeted with the file transfer example UI. Awesome!

--- INSET SCREENSHOT HERE --- 

Now, lets hit that `start IPFS` button on both of those tabs. We should get an ipfs id, as well as a `p2p-circuit` address under the `Addresses` section

-- INSERT SCREENSHOT HERE ---

Lets get the browser nodes connected to our relay node. Remember that address we noted above, the `/ip4/127.0.0.1/tcp/4003/ws/ipfs/...`, lets paste it into the text box under the `Peers` section in each of the browser tabs and hit `Connect to peer`.

--- INSET SCREENSHOT HERE ---

We should se the address we just pasted appear right under the `Connect to peer` button. Great, we've just connected to the relay node!

Now, lets get our browser nodes connected. Remember, browsers cant connect to each other directly, but now that we got `circuit-relay` we can make that happen ;)! Lets pick one of those browser tabs (any one tab, it doesn't matter), right under the `ID` of our browser peer we should see the `Addresses` section

--- INSERT SCREENSHOT HERE ---

Copy it from the tab and lets head over to the other tab we've opened. Paste the address in the `Peers` text box and hit `Connect to peer`, you should see your other browser circuit address appear right under the relay address we previously connected too!

--- INSERT SCREENSHOT HERE ----

It's worth stopping for a second and realizing what just happened! We've just connected two browser nodes without using any specialized rendezvous points, just plain and simple ipfs nodes! Now that is something!

### 4. Transfer files between the two browser nodes

Let's see what we can do with it! We've used the `exchange-files-in-browser` for a reason, we can use it to demonstrate the power of `circuit-relay` in full. Lets get to it then! You can use any file of you're choosing, but we've conveniently provided an image with this example located under the `img` folder. Just as the file transfer interface says, we can drag it right on to the UI. We should get the hash of the image (or the file of your choosing) in the text box under the `Files` section in the tab we just dropped the file, if you're using the provided image, the hash should be `QmVBW6KMRcsRjCr2Mu8iX52fUruvWKqsQUg8azjMmkRVaB`.

---- INSERT SCREENSHOT HERE --- 

Now lets head over to the other tab and paste that hash into the `Files` box in that tab, hit the `Fetch` button and we should  get a listing under the `Fetch` button

--- INSERT SCREENSHOT HERE ---

Lets click the hash under the `CID` column, the browser should initiate a download and we should get the file we just added in the other tab! This is nothing short of awesome!

### Conclusion...