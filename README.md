Loop Dropping Electron Madness
===

with **@MattMcKegg**

> Node.js Wellington Talk - 2 September 2015

## My **15 years** of **computer music**!

### **2001:** Steinberg Cubase LE

- came with our family computer!

### **2003:** Cool Edit Pro

### **2005-2007:** FL Studio

- the good old days
- released 2 albums online and got a little Internet famous as [Lunar](http://lunarmusic.net)

### **2008:** Ableton Live

### **2011**: NI Maschine

- some weird mash-up of Node + Maschine + Ableton Live
- played a few gigs as [Unidub](http://unidub.co.nz)

### **2013**: Started working on _Loop Drop_

- JavaScript + Web Audio

### **2015**: Loop Drop _finally_ started _not sucking_

## Loop Drop

A looper, modular synth and sampler designed for improvisation and live performance.

[http://loopjs.com](http://loopjs.com)

```bash
$ npm install loop-drop -g
$ loop-drop
```

> **Play a song**: [Quantum Loop?](https://www.youtube.com/watch?v=EBkmdNDIR6E)

## Electron

> Normally when I do a tech talk about Loop Drop, I talk about Web Audio + MIDI. But today, I'm gonna talk about the runtime I use to package Loop Drop as a desktop application.

Build cross-platform desktop applications in JavaScript and HTML+CSS.

[http://electron.atom.io](http://electron.atom.io)

## Atom

A text editor made by **GitHub** - powered by **electron**.

> Electron was originally called **atom-shell**. It was extracted out of Github's Atom text editor and open sourced long before Atom itself was opensource.

[http://atom.io](http://atom.io)

## Electron is **Node** (io.js) + **Shell** + **Chromium**

> Electron is a wrapper around **libchromiumcontent** (basically an embeddable version of Chrome - chromium _without_ the chrome)

> On the surface it works a lot like the Node executable.

```bash
$ npm install electron-prebuilt
$ electron main.js
```

> But it provides access to a bunch of special modules that allow creation of browser windows, access to desktop menus, tray icons, system dialogs and more.

### main.js

```js
console.log("I'm just a Node app. Right?")

var BrowserWindow = require('browser-window')
var Dialog = require('dialog')
console.log("Wrong. I have access to some very interesting modules!")

require('app').on('ready', function () {
  var window = new BrowserWindow({ width: 640, height: 480 })
  window.loadUrl('http://loopjs.com')

  Dialog.showOpenDialog(function (path) {
    console.log('You selected ' + path)
  })
})
```

## _Yo dawg,_ we put a **Node** in your **Browser**

> Not only can you spawn browsers with Electron, the spawned browsers themselves have access to node modules. It's like browserify without the build step. And also things like native addons and direct file system access via `require('fs')`.

```html
<html>
  <head>
    <title>An electron app window!</title>
  </head>
  <body>
    <h1>Hello, world!</h1>
    <script>
      var fs = require('fs')
      fs.writeFile('test.txt', 'Some text!', function (err) { })
    </script>
  </body>
</html>
```

## Let's build a thing!

> When I'm demoing Loop Drop, it's often hard for the audience to see what I'm doing on my hardware controllers. In the past I've often had a camera set up connected to a second projector.

A baby monitor for Launchpads.

> Today we're gonna solve the problem, and we're going to do it with Electron!

### main.js

> This is just electron boilerplate. It is responsible for creating the window.

```js
var join = require('path').join
var BrowserWindow = require('browser-window')
var app = require('app')

app.on('ready', function () {
  var window = new BrowserWindow({
    'title': 'Remote Camera',
    'width': 400,
    'height': 300
  })

  window.on('closed', function () {
    win = null
  })

  window.loadUrl('file://' + join(__dirname, 'index.html'))
  window.show()
})
```

### index.html

> In this file, we create a web server. It provides a website to the client and a websocket for them to connect to.

> [SimplePeer](https://github.com/feross/simple-peer) is a wrapper around [RTCPeerConnection](https://developer.mozilla.org/en-US/docs/Web/API/RTCPeerConnection).

> We are not actually streaming the webcam data via the websocket - we're just using web-rtc.

> [beefy](https://github.com/chrisdickinson/beefy) is an http server  that serves up the entries provided as a browserify bundle and can optionally provide a boiler-plate index.html.

```js
<html>
  <head>
    <link href='styles.css' rel='stylesheet' />
  </head>
  <body>
    <video autoplay id='viewer'></video>
    <script>
      var http = require('http')
      var beefy = require('beefy')
      var WebSocketServer = require('ws').Server
      var SimplePeer = require('simple-peer')
      var videoElement = document.getElementById('viewer')

      // serve the remote website to the client
      var server = http.createServer(beefy({
        entries: ['entry.js'],
        cwd: __dirname + '/client'
      }))

      // websocket for initiating web-cam stream
      var peerServer = new WebSocketServer({ server: server })
      peerServer.on('connection', function (socket) {
        var peer = new SimplePeer()
        peer.on('stream', function (stream) {
          // assign the remote web-cam stream to the video element
          videoElement.src = URL.createObjectURL(stream)
        })

        // forward the web-rtc peer signals between client/server
        socket.on('message', function (data) {
          peer.signal(JSON.parse(data))
        })
        peer.on('signal', function (data) {
          socket.send(JSON.stringify(data))
        })
      })

      server.listen(8080)
    </script>
  </body>
</html>
```

### client/entry.js

```js
var getUserMedia = require('getusermedia')
var SimplePeer = require('simple-peer')
var WebSocket = global.WebSocket

getUserMedia({video: true}, function (err, stream) {
  if (err) throw err
  var server = new WebSocket('ws://' + window.location.host)
  server.onopen = function () {
    start(stream, server)
  }
})

function start (stream, server) {
  var peer = new SimplePeer({ initiator: true, stream: stream })

  server.onmessage = function (e) {
    peer.signal(JSON.parse(e.data))
  }

  peer.on('signal', function (data) {
    server.send(JSON.stringify(data))
  })
}
```

### package.json

```js
{
  "scripts": {
    "start": "electron main.js"
  },
  "dependencies": {
    "beefy": "~2.1.5",
    "browserify": "~11.0.1",
    "electron-prebuilt": "~0.31.1",
    "getusermedia": "~1.3.5",
    "simple-peer": "~5.11.5",
    "ws": "~0.8.0"
  }
}
```

## And here's one I prepared earlier

[http://github.com/mmckegg/remote-camera](http://github.com/mmckegg/remote-camera)

## Enough code, let's make some music!

> Now everyone can see what I'm doing :)

[http://github.com/mmckegg/loop-drop-app](http://github.com/mmckegg/loop-drop-app)

## Sampling the _finest_ internets

[https://gist.github.com/mmckegg/4f39a109b833572be57a](https://gist.github.com/mmckegg/4f39a109b833572be57a)

## You can contribute!

- install the app from [npm](https://www.npmjs.com/package/loop-drop), and tell me what you think!
- **clone the repo** and contribute _more controller mappings_!
- or just buy the app :)

## DESTROY WITH SCIENCE

[soundcloud.com/destroy-with-science](https://soundcloud.com/destroy-with-science)

## Thanks for listening!

**@MattMcKegg**

[github.com/mmckegg/nodejs-wellington-talk-september-2015](https://github.com/mmckegg/nodejs-wellington-talk-september-2015)
