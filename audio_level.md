# Audio Level for encoded RTC media frames

## Authors:

- Guido Urdaneta (Google)

## Participate
- https://github.com/w3c/webrtc-encoded-transform


## Introduction

The [WebRTC Encoded Transform](https://w3c.github.io/webrtc-encoded-transform/)
API allows applications to access encoded media flowing through a WebRTC
[RTCPeerConnection](https://w3c.github.io/webrtc-pc/#dom-rtcpeerconnection). 
Video data is exposed as 
[RTCEncodedVideoFrame](https://w3c.github.io/webrtc-encoded-transform/#rtcencodedvideoframe)s
and audio data is exposed as
[RTCEncodedAudioFrame](https://w3c.github.io/webrtc-encoded-transform/#rtcencodedaudioframe)s.
Both types of frames have a getMetadata() method that returns a number of
metadata fields containing more information about the frames.

This feature consists in adding an additional metadata field called `audioLevel`, 
containing the audio level to RTCEncodedAudioFrame. This metadata field is
defined as follows:
* Represents the audio level of the frame. The value is between 0..1 (linear), 
where 1.0 represents 0 dBov, 0 represents silence, and 0.5 represents approximately
6 dBSPL change in the sound pressure level from 0 dBov.

For incoming frames, the value is obtained from the Audio Level header extension [RFC 6464](https://www.rfc-editor.org/rfc/rfc6464) and [RFC 6565](https://www.rfc-editor.org/rfc/rfc6465).

Note that the [RTCRtpContributingSource](https://w3c.github.io/webrtc-pc/#dom-rtcrtpcontributingsource-audiolevel) 
dictionary and [WebRTC stats](https://w3c.github.io/webrtc-stats/#dom-rtcinboundrtpstreamstats-audiolevel) also exposes this value,
but in a way that is not suitable for applications using the WebRTC Encoded
Transform API. The reason is that encoded transforms operate per frame, while
the values the existing APIs are the most recent seen by the UA,
which make it impossible to know for sure that their values actually correspond
to the frames being processed by the application using WebRTC Encoded Transform.


## User-Facing Problem

This API supports applications where knowing the audio level of an audio frame
is useful.

Some examples use cases are:
1. Showing an indication that a participant in a videoconference is speaking.
2. Reduce redundancy for audio frames with zero/low audio level.


### Goals

- Provide Web applications using WebRTC Encoded Transform access to audio level
  in addition to existing metadata already provided.


### Non-goals

- Provide mechanisms to improve WebRTC communication mechanisms based on the
information provided by these new metadata fields.


### Example

This shows an example of an application that:
1. Updates a speaking indication.
2. Drops audio frames with zero audio level.

```js
function updateUI(audioLevel) {
  postMessage(audioLevel);
}

// code in a DedicatedWorker
function createReceiverAudioTransform() {
  return new TransformStream({
    start() {},
    flush() {},
    async transform(encodedFrame, controller) {
      let metadata = encodedFrame.getMetadata();
      updateUI(metadata.audioLevel);
      // drop frames with zero audioLevel
      if (metadata.audioLevel > 0.0) {
        controller.enqueue(encodedFrame);
      }
    }
  });
}


// Code to instantiate transforms and attach them to sender/receiver pipelines.
onrtctransform = (event) => {
  let transform;
  if (event.transformer.options.name == "receiverAudioTransform")
    transform = createReceiverAudioTransform();
  else
    return;
  event.transformer.readable
      .pipeThrough(transform)
      .pipeTo(event.transformer.writable);
};

// Code running on Window
const worker = new Worker('worker.js');
const pc = new RTCPeerConnection();

// Do ICE and offer/answer exchange. Removed from this example for clarity.

// Configure transforms in the worker
pc.ontrack = e => {
  if (e.track.kind == "audio")
    e.receiver.transform = new RTCRtpScriptTransform(worker, { name: "receiverAudioTransform" });
}

// Handle audio-level updates
worker.onmessage = e => {
  DoUpdateUI(e.data);
}
```


## Alternatives considered

### [Alternative 1]

Use the values already exposed in `RTCRtpContributingSource` or stats.

`RTCRtpContibutingSource` and WebRTC stats already expose the audioLevel.
The problem with using those timestamps is that it is impossible to reliably
associate them to a specific encoded frame exposed by the WebRTC Encoded
Transform API.

This makes any of the computations in this feature unreliable.
Moreover, using these values requires polling, which is less efficient than
reading the value directly from the frame.



## Accessibility, Privacy, and Security Considerations

The audioLevel is already available in a form less suitable for applications
using WebRTC Encoded Transform as part of the RTCRtpContributingSource API and
[WebRTC Stats](https://w3c.github.io/webrtc-stats/#dom-rtcinboundrtpstreamstats-audiolevel).

While these fields are not 100% equivalent to the fields in this feature,
they have the same privacy characteristics. Therefore, we consider that the
privacy delta of this feature is zero.
