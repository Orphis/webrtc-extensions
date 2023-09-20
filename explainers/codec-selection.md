# WebRTC Codec Selection API

## Authors:

- Florent Castelli, Google
- Henrik Boström, Google

## Participate
- [Issue tracker](https://github.com/w3c/webrtc-extensions/issues/)

## Introduction

WebRTC implementations use by default the first negotiated codec for each RTP stream. While this behavior is good for compatibility and simplicity, advanced scenarios where the applications want a finer control over the codec used and the encoding parameters become complicated and not possible.

While the current APIs allow an application to change the codec order, renegotiate and update the encoding parameters, this will require many calls and will possibly start sending media with the new codec but old parameter for a short moment. Different codecs are likely to support a different set of scalability modes, so being able to configure both the codec and its scalability mode in the same API call is essential to being able to reconfigure the encoder smoothly, without temporarily using a wrong configuration or producing unnecessary key frames due to multiple reconfiguration steps or having to temporarily disable the stream during reconfiguration.

Additionally, since only the first codec is used for all the RTP streams on a single transceiver object, it is not possible to use a different codec on simulcast layers. With the emergence of newer codecs with better efficiency, this configuration has become more attractive for applications.

## Goals

The goal of the proposed API is to allow an application to update an RTP stream to use any negotiated codec and update the encoding parameters atomically using familiar APIs. The API should work for both audio and video media content.

## Non-goals

The proposed API does not intend to allow creating of changing the codec properties or creating new entries and should only work with existing negotiated codecs or codecs part of the codec capabilities.

## RTCRtpEncodingParameters.codec

```js
// Convenience function
function findVideoCodec(name) {
    return RTCRtpSender.getCapabilities('video').codecs.filter(c => c.mimeType == name)[0];
}

// Changes the codec on the first sender to AV1 and make it use the L3T3 SVC mode.
const pc = new RTCPeerConnection();
const transceiver = pc.addTransceiver('video');
const sender = transceiver.sender;

const parameters = sender.getParameters();
// Using the new encoding parameters to change the current codec to AV1 and set the scalability mode in a single call.
// See https://w3c.github.io/webrtc-extensions/#dom-rtcrtpencodingparameters-codec
parameters.encodings[0].codec = findVideoCodec('video/AV1');
parameters.encodings[0].scalabilityMode = 'L3T3';

// See https://w3c.github.io/webrtc-extensions/#setparameters for the extra validation steps.
await sender.setParameters(parameters);

// Creates a new transceiver with mixed-codec simulcast using VP8 and AV1 and different scalability modes.
// See https://w3c.github.io/webrtc-extensions/#addtransceiver for the extra validation steps.
const mixedCodecTransceiver = pc.addTransceiver('video', {
  sendEncodings: [
    {
        codec: findVideoCodec('video/VP8'),
        scalabilityMode: 'L1T3',
        scaleResolutionDownBy: 1
    },
    {
        codec: findVideoCodec('video/AV1'),
        scalabilityMode: 'L3T3',
        scaleResolutionDownBy: 1
    }]
});
```

## Detailed design discussion

### Error handling

Using a new codec or a codec with different parameters is not supported by this API.

Requested codecs are checked against the list of supported codecs before negotiation and against the list of negotiated codecs after negotiation. If the codecs are not found, the call to `addTransceiver()` or `setParameters()` fails.

```js
// Throws OperationError
const mixedCodecTransceiver = pc.addTransceiver('video', {
  sendEncodings: [
    {
        codec: findVideoCodec('video/NewCodec'),
        scalabilityMode: 'L1T3',
        scaleResolutionDownBy: 1
    }]
});
```

### Renegotiation

Negotiating away a codec that is currently selected is another tricky scenario. A renegotiation could remove a currently selected codec from the list of negotiated codecs for the transceiver.

This was decided not to be an error, but instead reset the selected codec value for the user-agent to pick a codec automatically and ensure that media always flows.

## Considered alternatives

### RTCRtpEncodingParameters.payloadType

Older versions of the WebRTC specification had an extra dictionary member `payloadType` that would allow to force a codec in a similar way to this proposed API as well as get the payload type value for the currently used codec for the matching RTP stream.

This caused issues as you could only use this after negotiation and never with `addTransceiver()`. Parameters would also be updated asynchronously and interfere with the get-modify-set pattern from `getParameters()` and `setParameters()`.

## Stakeholder Feedback / Opposition

- Google : Positive
- WebKit : [Positive](https://github.com/WebKit/standards-positions/issues/179)
- Mozilla : [Positive](https://github.com/mozilla/standards-positions/issues/789)
