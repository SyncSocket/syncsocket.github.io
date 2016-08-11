---
layout: page
title: Get started
permalink: /get-started/
order: 1
---

In this guide we will build a simple web-based multi-room audio system, that can play music through multiple seperate computers.

### Introduction

Writing an application that involves making multiple computers doing something synchronously (i.e. at exactly the same moment) can be quite challenging, especially when computers are connected over the internet, since you can't predict how long it would take for a message to be sent from your server to target host. Depending on particular problem you are solving, this may be not a problem at all, however, in some cases you may want to ensure that connected devices are actually doing things at once.

For example, for a chat application there's probably no need for chat messages to be delivered to recepient's phone and tablet at exactly same moment — delay may get quite long but user experience won't get bad. In this case you don't need synchronized messaging, regular messaging will work. If it's your case, you can take a look at [socket.io](http://socket.io).

However, when you really need perform an action at the same moment, you need synchronization. For example, when you want to play audio or video content on multiple devices together. If one of your devices starts playback as little as 50 milliseconds after others, it will introduce a nasty echo-like effect to the music and ruin the playback. SyncSocket does just that, it allows you to trigger an action at the same moment.

### The web server

Let's begin with setting up a webpage that will serve as playback control for your multiroom system. To keep things simple, we will use `express` web framework for that. Make sure that you have [node.js](http://nodejs.org) installed.

Begin with creating an empty folder for our app (for example, `multiroom-audio-example`), and then create a file `package.json` in it that will describe our project

```json
{
    "name": "multiroom-audio-example",
    "version": "0.0.1",
    "description": "my first SyncSocket app"
}
```

Next, we need to install `express` as our project dependency

```
npm install express --save
```

Now express is installed and we can start writing code for our app. Create file `index.js` with following contents

```js
var app = require('express')();
var http = require('http').Server(app);

app.get('/', function(req, res){
  res.send('<h1>Hello world</h1>');
});

http.listen(3000, function(){
  console.log('listening on *:3000');
});
```

Let's see what the code does.

1.  It declares variable `app` to handle HTTP requests to the app.
2.  Defines a GET request handler for path `/` that will send HTML `<h1>Hello world</h1>` to requesting user
3.  Instructs the HTTP server to listen for incoming requests on port 3000

You can run the app by typing `node index.js` and you should see a message that server is listening on port 3000.
Then in your browser type [http://localhost:3000](http://localhost:3000) and you should see the Hello World message displayed

![Hello world in browser]({{ site.url }}/assets/hello-browser.png)

### The player UI

Lets instead of sending a HTML string send a contents of a whole HTML file, replacing `res.send` with `res.sendFile` in file `index.js`

```js
app.get('/', function(req, res){
  res.sendFile(__dirname + '/index.html');
});
```

Now create a file `index.html` that will draw our player user interface

```html
<!doctype html>
<html>
    <head>
        <title>SyncSocket Multiroom Audio</title>
    </head>
    <body>
    <center>
        <button id="play-btn">Play</button>
        <button id="stop-btn">Stop</button>
    </center>
        <center>
            <p>Message: <b><span id="message">no message</span></b></p>
        </center>
    </body>
</html>
```

Now restart the server (by hitting Control-C and typing `node index.js` again) and then refresh the page. You should see something like this

![Basic player]({{ site.url }}/assets/player-browser.png)

### Integrating SyncSocket

SyncSocket framework consists of two parts:

1.  The server part handles all messages and syncs devices: `syncsocket`
2.  The client, that connects to server and can send and receive messages: `syncsocket-client`

First let's integrate SyncSocket server to our app. We need to install it

```
npm install syncsocket --save
```

Now let's setup our SyncSocket server to run besides the web server. Add the following to `index.js`

```js
var SyncSocket = require('syncsocket');

var syncsocketSrv = SyncSocket();
syncsocketSrv.createChannel({ channelId: 'myAudioSystem' });
syncsocketSrv.listen(6024);
```

This code:

1.  Creates SyncSocket server object
2.  Creates a channel with id `my-audio-system` that your clients will join. A channel is like a radio station: when you send a message to channel, all connected clients will receive this message synchronously
3.  Instructs SyncSocket server to listen for clients on port `6024`.

Second, lets integrate SyncSocket-client into our website code.

```
npm install syncsocket-client --save
```

The client is now installed into `node_moules/syncsocket-client` folder. Now we need to make our web server to serve the SyncSocket client script besides HTML. So add another handler to `index.js`

```js
app.get('/syncsocket-client.js', function (req, res) {
    res.sendFile(__dirname + '/node_modules/syncsocket-client/syncsocket.js')
});
```

Also we wish to know when a new client connects to our server. We can use `connected` event for that: it will fire executing the handler every time someone connects. Let's add it to `index.js`

```js
syncsocketSrv.on('connection', function () {
    console.log('a new client connected');
});
```

Don't forget to restart the server to make changes effective. Next step is to use the client to connect to our server. Add the following to `index.html`

```html
<script src="/syncsocket-client.js"></script>
<script>
    var client = syncsocket('http://localhost:6024');
</script>
```

Now try to refresh the page a couple of times. You should see the messages about new clients connected in the server window:

![Clients connect]({{ site.url }}/assets/srv-client-connected.png)

There is also another event that fires when a client disconnects. Add this to `index.js`

```js
syncsocketSrv.on('disconnect', function () {
    console.log('client disconnected');
});
```

Now restart server and refresh page a couple of times. You should see something like this

![Clients connect disconnect]({{ site.url }}/assets/srv-client-dc.png)

### Joining a channel

Remember we created a channel that works like radio station? Now let client join that channel. Edit `index.html` to add new instructions

```html
<script>
    var client = syncsocket('http://localhost:6024');
    client.on('connected', function () {
        client.joinChannel('myAudioSystem')
            .then(function (channel) {
                channel.on('syncSuccessful', function () {
                    document.getElementById('message').innerHTML = "joined and synchronized channel!";
                });
            });
    });
</script>
```

Refresh the page and you should see the message saying that channel is joined and synchronized.

![Joined and synchronized]({{ site.url }}/assets/joined-and-sync.png)

### Subscribing and publishing channel messages

In channels, there are two important methods. They allow you to send and receive messages from channels.

1. `publish(topic, data)` allows you to publish a message to the channel on given topic.
2. `subscribe(topic, prepareCallback, fireCallback)` allows you to subscribe and start receiving messages on given topic.

The first one is quite obvious. You publish a message which has two parts: a topic, that identifies type of the message, and an arbitrary object data.
The second one is more tricky: when subscribing to messages with given topic, you should provide two callbacks, as clients receive messages **twice**.
First callback is your chance to prepare for an upcoming event. This is needed to perform all the heavy-lifting in order to execute event quickly. In our example,
we may want to preload the music, pre-fill audio buffer, i.e. do everything we can to begin playing music as quick as possible during actual event.
Second callback is when you actually need to execute event that you've prepared in first callback.

Now let's change our `index.html` to try out this publish-subscribe mechanism

```html
<script>
        function setMessage(message) {
            document.getElementById('message').innerHTML = message;
        }
        var client = syncsocket('http://localhost:6024');
        client.on('connected', function () {
            client.joinChannel('myAudioSystem', true)
                    .then(function (channel) {
                        channel.on('syncSuccessful', function () {
                            setMessage("joined channel and synchronized!");

                            // Play button click handler
                            document.getElementById('play-btn').onclick = function () {
                                channel.publish('play');
                            };

                            channel.subscribe('play',
                                    function () { // Prepare callback
                                        setMessage("preparing to play...");
                                    },
                                    function () { // Fire callback
                                        setMessage("playing music");
                                    }
                            );
                        });
                    });
        });
</script>
```

Notice that we do two new things here: first we set click handler for the `Play` button. When user clicks the button, a message with topic `play` is published to channel.
Second, we subscribe our channel to messages with topic `play`. The prepare callback displays message `prepare to play...`, and the fire callback displays `playing music`.

Now if you open two browser windows and place them side by side, you should see that both clients can send and receive messages like this

![Synced]({{ site.url }}/assets/music-demo-1.gif)

### Adding some music

Let's finally add some music so that we can hear the synchronous playback ourselves.
We should change the code in prepare callback to the following

```js
setMessage("preparing to play...");
var context;
try {
    window.AudioContext = window.AudioContext || window.webkitAudioContext;
    context = new AudioContext();
} catch (e) {
    alert('Web Audio API is not supported in this browser');
}
source = context.createBufferSource();
var request = new XMLHttpRequest();
request.open('GET', 'https://dl.dropboxusercontent.com/u/752038/sample2.mp3', true);
request.responseType = 'arraybuffer';
request.onload = function () {
    var audioData = request.response;
    context.decodeAudioData(audioData, function (buffer) {
        source.buffer = buffer;
        source.connect(context.destination);
        channel.finalizeTransition();
    }, function (err) {
        throw "Error decoding audio data: " + err;
    });
};

channel.deferTransition();
request.send();
```

This code does the following.

1. It creates an AudioContext, which is required for playing sounds via WebAudio API.
2. Creates a BufferSource, which is a chunk of memory into which we will load our audio data.
3. Issues a request to fetch an MP3 file which we are going to play.
4. Uncompresses (decodes) MP3 data and stores result in a buffer
5. Connects destination (i.e. your speakers) to the buffer with decoded audio.

Couple of things to note.
Observe, that before sending the request for audio data, we call `channel.deferTransition` function.
This will pause the preparing process until `channel.finalizeTransition` is called (notice that we call
it after we've loaded and decoded data, etc). This pattern can be really helpful for asynchronous operations (just as we have here with HTTP request). If we omit these two calls, then client will finish preparation just after the request to fetch audio is sent, and may get into situation when `play` event is fired, but music is not yet loaded.

Now let's change the fire callback

```js
source.start(0);
setMessage("playing music");
```

This is easy: we simply tell the source to start playing audio now.
Now if we open two browser windows side-by-side and hit `Play`, we should hear same music playing from both windows at the same time.

### Homework problems

+   Make `Stop` button work. Clicking it should make all playing browsers to pause music at the same time.
+   Let user to choose among two or three audio files to play. When publishing `play` message, pass along some data that will identify the choice. Don't use different message topics, use extra data.
+   Use another computer instead of second browser window to serve as speaker in your multi-room system. **Hint**: by default, clients attempt to connect and synchronize with `localhost`. This works on your local machine, but for remote clients it would not. You need to determine your machine's IP, and then pass it to server constructor spec as defaultTimeserver with port 5579 (e.g. `var syncsocketSrv = SyncSocket({ defaultTimeserver: 'http://192.168.1.28:5579' })`). Also you'll need to change the URI the clients connect to -- use port 6024 (e.g. `var client = syncsocket('http://192.168.1.28:6024')`)

### Getting example source code

Head over to the [GitHub repo](https://github.com/woyorus/multiroom-audio-example), or

```shell
git clone https://github.com/woyorus/multiroom-audio-example.git
```
