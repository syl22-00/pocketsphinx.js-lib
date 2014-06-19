PocketSphinx.js bower package
-----------------------------

This is the bower package for the [PocketSphinx.js](https://github.com/syl22-00/pocketsphinx.js) library. It is typically installed as a dependency of other bower packages, such as acoustic or language models for PocketSphinx.js.

Here we document the general usage of PocketSphinx.js, each acoustic and language model package also ships with its own documentation. Also, refer to the source repository for a more comprehensive documentation.

# 1. What is PocketSphinx.js?

PocketSphinx.js is a JavaScript library that does speech recognition. It records audio and recognizes spoken words in real time. Recording and recognition are done entirely in JavaScript, there is no server-side processing or browser plug-in, everything happens in the client.

# 2. Usage

In addition to that library, in order to recognize speech you need, for the language you want to recognize:

* An acoustic model that describes how phonemes sound. There are bower packages for several acoustic models, matching several languages. You can also make your own (see source repository).
* A pronunciation dictionary that describes how phonemes combine into words. There are some dictionaries available as bower packages for several languages. Dictionary words can also be added at runtime, through the JavaScript API.
* A language model that describes how words combine into sentences. There are two types of language models: grammars and statistical language models. Grammars are used for constrained, small vocabulary tasks (such as voice commands) and can be added at runtime through the JavaScript API. Statistical language models are used for less contrained tasks, such as dictation. They must be loaded at init time, there are models available as bower packages for several languages.

To recognize speech in real time, you need to interact with both the recognizer and the audio recorder.

This documentation is a work in progress, see template.html for an example.

## 2.1 Recognizer

The recognition library has two main files, `recognizer.js` and `pocketsphinx.js` which are loaded and executed inside a Web worker. Once installed, they are located in `bower_components/pocketsphinx.js-lib/`. The entry point is `recognizer.js` which must be loaded as a new Web worker object:

```javascript
var recognizer = new Worker("bower_components/pocketsphinx.js-lib/recognizer.js");
```

You can then interact with it using messages.

### 2.1.1 Incoming Messages

Messages posted to the recognizer worker might include the following attributes:

* `command`, command to be executed,
* `data`, data to be passed to the command,
* `callbackId`, id to be passed to the outgoing message, might be used to trigger a callback.

### 2.1.2 Outgoing Messages

The worker sends messages back to the UI thread, either to call back when actions have been performed, report errors or send periodic information such as the current recognition hypothesis.

Messages posted by the recognizer worker might include:

* `status`, which can be either `done` or `error`,
* `command`, the command that sent the message,
* `code`, an error code,
* `id`, a callback id that was given in the received incoming message,
* `data`, additional data that the callback function might make use of,
* `hyp`, the current recognition hypothesis,
* `hypseg`, the current segmentation,
* `final`, a boolean that indicates whether the hypothesis is final (sent after call to `stop`).

### 2.1.3 API description

#### a. Error codes

The error codes returned in messages posted back from the worker can be:

* the error code returned by `pocketsphinx.js` as explained previously,
* or one of the following strings:
    * "js-data", if the provided data are invalid,
    * "js-no-recognizer", if the recognizer is not initialized.

#### b. Initialization

Once the worker is created, the recognizer must be initialized:

```javascript
// This value will be given in the message received after the action completes:
var id = 0;
recognizer.postMessage({command: 'initialize', callbackId: id});
```

Once it is done, the recognizer will post a message back, for instance:

* `{status: "done", command: "initialize", id: clbId}`, if successful, where `clbId` is the callback id given in the original command message.
* `{status: "error", command: "initialize", code: initStatus}`, if there is an error, where `initStatus` is the value returned by the call to `psInitialize`, see above for possible values.

Recognizer parameters to be passed to `PocketSphinx` can be given in the call to `initialize`. For instance:

```javascript
recognizer.postMessage({command: 'initialize',
                        callbackId: id,
                        data: [["-hmm", "french"],
                               ["-fwdflat", "no"],
                               ["-dict", "french.dic"],
                               ["-lm", "french.DMP"]]
                       });
```

This will set the `pocketsphinx` command-line parameter `-fwdflat` to `no` and initialize the recognizer with the acoustic model `french`, the language model `french.DMP` and the dictionary `french.dic`, assuming `pocketsphinx.js` was compiled with such models.

Note that once it is initialized, the recognizer can be re-initialized with different parameters. That way, for instance, a web application can switch between different acoustic and language models at runtime.

#### c. Adding words

Words to be recognized must be added to the recognizer before they can be used in grammars. See previous sections to know more about the format of dictionary items. Words can be added at any time after the recognizer is initialized, and several words can be added at once:

```javascript
// An array of pairs [word, pronunciation]:
var words = [["ONE", "W AH N"], ["TWO", "T UW"], ["THREE", "TH R IY"]];
recognizer.postMessage({command: 'addWords', data: words, callbackId: id});
```

The message back could be:

* `{id: clbId}`, the provided callback id, if given, as explained before, if successful.
* `{status: "error", command: "addWords", code: code}`, if error, where possible values of the error code was described above.

Note that words can have several pronunciation alternatives.

#### d. Adding grammars or key phrases

As described previously, any number of grammars or keyword searches can be added. The recognizer can then switch between them. 


A grammar can be added at once using a JavaScript object that contains the number of states, the first and last states, and an array of transitions, for instance:

```javascript
var grammar = {numStates: 3,
               start: 0,
               end: 2,
               transitions: [{from: 0, to: 1, word: "HELLO"},
                             {from: 1, to: 2, logp: 0, word: "WORLD"},
                             {from: 1, to: 2}]
              };
recognizer.postMessage({command: 'addGrammar', data: grammar, callbackId: id});
```

All words must have been added previously using the `addWords` command.

Notice that `logp` is optional, it defaults to 0. `word` is also optional, it defaults to `""` which is a null-transition.

In the message back, the grammar id assigned to the grammar is given. It can be used to switch to that grammar. So the message, if successful, would be like `{id: clbId, data: id, status: "done", command: "addGrammar"}`, where `id` is the id of the newly created grammar. In case of errors, the message would be as described previously.

Similarly, keyword spotting search can be added by just providing the key phrase to spot:

```javascript
var keyphrase = "HELLO WORLD";
recognizer.postMessage({command: 'addKeyword', data: keyphrase, callbackId: id});
```

Just as like with grammars, words should already be in the recognizer, and the id of the newly added search is given in the callback. As explained previously, you might want to ajust the sensitivity threshold when initializing the recognizer, for example with providing `["-kws_threshold", "2"]`.


#### e. Starting recognition

The message to start recognition should include the id of the grammar (or keyword search) to be used:

```javascript
// id is the id of a previously added grammar:
recognizer.postMessage({command: 'start', data: id});
```

#### f. Processing data

Audio samples should be sent to the recognizer using the `process` command:

```javascript
// array is an array of audio samples:
recognizer.postMessage({command: 'process', data: array});
```
Audio samples should be 2-byte integers, at 16 kHz.

While data are processed, hypothesis will be sent back in a message in the form `{hyp: "RECOGNIZED STRING"}`. If it is a keyword spotting search, the `hyp` field will be the key phrase, present as many times as it appeared since recognition started.

#### g. Ending recognition

Recognition can be simply stopped using the `stop` command: 

```javascript
recognizer.postMessage({command: 'stop'});
```

It will then send a last message with the hypothesis, marked as final (which means that it is more accurate as it comes after a second pass that was triggered by the `stop` command). It would look like: `{hyp: "FINAL RECOGNIZED STRING", final: true}`.

#### h. Loading files (such as acoustic models packaged outside `pocketsphinx.js`

The recognizer worker can load any JavaScript file. This is especially useful to load the files of acoustic models, language models or dictionaries, so that they are available when the recognizer is initialized or re-initialized with specific values of `-hmm`, `-lm` or `-dict`.

```javascript
recognizer.postMessage({command: 'load',
                        callbackId: id,
                        data: ["mymodel/mdef.js",
                               "mymodel/transition_matrices.js",
                               ...
                               "mymodel/variances.js"]
                       });
```

The path is relative to the location of `recognizer.js`, and in this example, the model can be loaded by using a `config` object with `["-hmm", "mymodel"]`.

There will be an error callback with `NETWORK_ERROR` if any of the files can't be loaded.

## 2.2 Using `CallbackManager`

In order to facilitate the interaction with the recognizer worker, we have made a simple utility that helps associate callbacks to be executed when the worker posts a message responding to a command you sent. You can find `callbackManager.js` in `bower_components/pocketsphinx.js-lib/`.

To use it, first create a new instance of CallbackManager:

```javascript
var callbackManager = new CallbackManager();
```

When you post a message to the recognizer worker and want to associate a callback function to it, you first add your callback function to the manager, which gives you a callback id in return:

```javascript
recognizer.postMessage({command: 'addWords',
                        data: words,
                        callbackId: callbackManager.add(
                           function() {alert("Words added");})
                       });
```

In the `onmessage` function of your worker, use the callback manager to check and trigger callback functions:

```javascript
recognizer.onmessage = function(e) {
    if (e.data.hasOwnProperty('id')) {
        // If the message has an id field, it
        // means that we might have a callback associated
        var clb = callbackManager.get(e.data['id']);
        var data = {};
        // As mentioned previously, additional data can be passed to the callback
        // such as the id of a newly added grammar
        if(e.data.hasOwnProperty('data')) data = e.data.data;
        if(clb) clb(data);
    }
    // Check for other message types here
};
```

## 2.3 Detecting when the worker is ready

When a new worker is instantiated, it immediately returns a worker object, but the actual download of the JavaScript files might take some time, especially in our case where `pocketsphinx.js` is fairly large. One way of detecting whether the files are fully downloaded and loaded is to post a first message right after it is instantiated and wait for a message back from the worker.

````javascript
var recognizer;
function spawnWorker(workerurl, onReady) {
    recognizer = new Worker(workerurl);
    recognizer.onmessage = function(event) {
        // onReady will be called when there is a message
        // back
        onReady(recognizer);
    };
    recognizer.postMessage('');
};
```

The first message posted to the recognizer can include the name of the PocketSphinx JavaScript file to load. This is handy if you want to build an application with several different models, you can keep the same `recognizer.js` file for different parts of your application and load any PocketSphinx JavaScript file that you want. By default, it will load `pocketsphinx.js`, but if you want your application to load a file called `pocketsphinx_chinese.js`, you can just add it as parameter to the first posted message:

````javascript
var recognizer;
function spawnWorker(workerurl, onReady) {
    recognizer = new Worker(workerurl);
    recognizer.onmessage = function(event) {
        // onReady will be called when there is a message
        // back
        onReady(recognizer);
    };
    recognizer.postMessage('pocketsphinx_chinese.js');
};
```

After the first message back was received, proper listening to onmessage can be added:

```javascript
spawnWorker("js/recognizer.js", function(worker) {
    worker.onmessage = function(e) {
    // Add what you want to do with messages back from the worker
    };
    // Here is a good place to send the 'initialize' command to the recognizer
});
```

Of course, the worker must be able to respond to the first message, as we did in `recognizer.js`:

```javascript
function startup(onMessage) {
    self.onmessage = function(event) {
        self.onmessage = onMessage;
        self.postMessage({});
    }
};
// This function is called first, it triggers
// a first postmessage, then adds the proper respond to
// commands: 
startup(function(event) {
    switch(event.data.command){
        //We deal with commands properly
    }
});
```

All these are illustrated in `template.html`.

# 5. Wiring `recognizer.js` to the audio recorder

We include an audio recording library based on the Web Audio API that accesses the microphone, gets audio samples, converts them to the proper sample rate (16kHz for our default acoustic model), and sends them to the recognizer. This library is derived from [Recorderjs](https://github.com/mattdiamond/Recorderjs). To know more about audio capture and playback on the web, you could have a look at this [overview of audio on the Web](https://github.com/syl22-00/TechDocs/blob/master/AudioInBrowser.md). A more complete documentation of the recorder can be found in `doc/AudioRecorder/README.md`.

Include `audioRecorder.js` in the HTML file and make sure `audioRecorderWorker.js` is in the same folder. To use it, create a new instance of `AudioRecorder` giving it as argument a `MediaStreamSource`. As of Today, the Google Chrome and Firefox (25+) implement it. You also need to set the recognizer attribute to a Recognizer worker, as described above.

```javascript
// Deal with prefixed APIs
window.AudioContext = window.AudioContext || window.webkitAudioContext;
navigator.getUserMedia = navigator.getUserMedia ||
                         navigator.webkitGetUserMedia ||
                         navigator.mozGetUserMedia;

// Instantiating AudioContext
try {
    var audioContext = new AudioContext();
} catch (e) {
    console.log("Error initializing Web Audio");
}

var recorder;
// Callback once the user authorizes access to the microphone:
function startUserMedia(stream) {
    var input = audioContext.createMediaStreamSource(stream);
    recorder = new AudioRecorder(input);
    // We can, for instance, add a recognizer as consumer
    if (recognizer) recorder.consumers.push(recognizer);
  };

// Actually call getUserMedia
if (navigator.getUserMedia)
    navigator.getUserMedia({audio: true},
                            startUserMedia,
                            function(e) {
                                console.log("No live audio input in this browser");
                            });
else console.log("No web audio support in this browser");
```

Once the recorder is up and running, you can start and stop recording and recognition with:

```javascript
// To start recording:
recorder.start();
// The hypothesis is periodically sent by the recognizer, as described previously
// To stop recording:
recorder.stop();  // The final hypothesis is sent
```

The constructor for AudioRecorder can take an optional config object. This config can include a callback function which is executed when there is an error during recording. As of today, the only possible error is when the input samples are silent. It can also include the output sample rate, which you might need to set if you use an acoustic model of 8kHz audio.

```javascript
var audioRecorderConfig = {
    errorCallback: function(x) {alert("Error from recorder: " + x);},
    outputSampleRate: 8000
    };
recorder = new AudioRecorder(input, audioRecorderConfig);
```

All these are illustrated in the given example, `template.html`.

Note that live audio capture is only available on recent versions of Google Chrome and Firefox. Chrome, prior to version 29, only produced silent audio on many platforms. Firefox includes the necessary features starting from version 25.




# License

PocketSphinx licensing terms are included in the `pocketsphinx` and `sphinxbase` folders of PocketSphinx.js source repository.

The files `audioRecorder.js` and `audioRecorderWorker.js` are based on [Recorder.js](https://github.com/mattdiamond/Recorderjs), which is under the MIT license (Copyright © 2013 Matt Diamond).

The remaining of this software is licensed under the MIT license:

Copyright © 2013-2014 Sylvain Chevalier

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
