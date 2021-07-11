<template>
  <div class="hello">
    <button @click="getRtpCapabilities">join</button>
    <video id="video"></video>
  </div>
</template>

<script>
import * as mediasoupClient from "mediasoup-client";
import * as axios from 'axios';

let device = new mediasoupClient.Device();

let peer = {
  peerId: '121212',
  sendTransport: null,
  recvTransport: null,
  producers:[],
  consumers:[],
}

async function getFromServer (url, data) {
  const response = await axios.post('http://localhost:3000' + url, {
    peerId: peer.peerId, ...data,
  });
  return response.data;
}

export default {
  name: 'HelloWorld',
  methods: {
    getRtpCapabilities: async function(){
      const {routerRtpCapabilities} = await getFromServer('/rtpCapabilities');

      console.log('load rtp cap from server', routerRtpCapabilities);
      await device.load({ routerRtpCapabilities });
     
      console.log('device load rtp cap');

      let localCam = await navigator.mediaDevices.getUserMedia({
        video: true,
        audio: true
      });
      console.log('get user media', localCam);


      let data1 = await getFromServer(`/create-transport/send/${peer.peerId}`);

      console.log('transportOptions', data1.transportOptions);
      peer.sendTransport = await device.createSendTransport(data1.transportOptions);

      peer.sendTransport.on('connect', async ({ dtlsParameters }, callback, errback) => {
        console.log('transport connect event', dtlsParameters, errback);

        let { error } = await getFromServer(`/connect-transport/send/${peer.peerId}`, {
          dtlsParameters,
        });

        if (error) {
          console.err('error connecting transport', error);
          errback();
          return;
        }
        callback();
      });

      peer.sendTransport.on('produce', async ({ kind, rtpParameters, appData },
                                   callback, errback) => {
        console.log('transport produce event', appData.mediaTag);
        let { error, id } = await getFromServer('/send-track/', {
          kind,
          rtpParameters,
          appData
        });
        if (error) {
          console.err('error setting up server-side producer', error);
          errback();
          return;
        }
        callback({ id });
      });

      const CAM_VIDEO_SIMULCAST_ENCODINGS =
      [
        { maxBitrate:  96000, scaleResolutionDownBy: 4 },
        { maxBitrate: 680000, scaleResolutionDownBy: 1 },
      ];

      function camEncodings() {
        return CAM_VIDEO_SIMULCAST_ENCODINGS;
      }

      let camVideoProducer = await peer.sendTransport.produce({
        track: localCam.getVideoTracks()[0],
        encodings: camEncodings(),
      });

      console.log('send ready', camVideoProducer);




      let {transportOptions} = await getFromServer(`/create-transport/recv/${peer.peerId}`);

      console.log('transportOptions', transportOptions);
      peer.recvTransport = await device.createRecvTransport(transportOptions);

      peer.recvTransport.on('connect', async ({ dtlsParameters }, callback, errback) => {
        console.log('transport connect event', dtlsParameters, errback);

        let { error } = await getFromServer(`/connect-transport/recv/${peer.peerId}`, {
          dtlsParameters,
        });

        if (error) {
          console.err('error connecting transport', error);
          errback();
          return;
        }
        callback();
      });


      let consumerParameters = await getFromServer(`/recv-track/${peer.peerId}`, {
        mediaTag: 'video',
        mediaPeerId: peer.peerId,
        rtpCapabilities: device.rtpCapabilities
      });

      console.log('consumer parameters', consumerParameters);
      let consumer = await peer.recvTransport.consume({
        ...consumerParameters,
        appData: { peerId: peer.peerId, mediaTag: 'video'}
      });

      console.log('consumer', consumer);

      let el = document.getElementById('video');
      el.srcObject = new MediaStream([ consumer.track.clone() ]);
      el.play();

      
    }
  },
  props: {
    msg: String,
  }
}
</script>
