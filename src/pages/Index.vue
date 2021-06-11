<template>
  <q-page>
    <div style="margin: 10px">
      <q-btn label="创建房间" v-on:click="createRoom"></q-btn> <br />
      <div class="row">
        <q-input v-model="room_id" label="当前房间号"></q-input>
        <q-btn label="加入房间" v-on:click="joinRoom"></q-btn>
      </div>
      <q-btn label="链接" v-on:click="start"></q-btn>
      <hr />
      <div>
        <video id="localVideo" autoplay=""></video>
        <video id="remoteVideo" autoplay=""></video>
      </div>
    </div>
  </q-page>
</template>

<script>
export default {
  name: "PageIndex",
  data() {
    return {
      ws: null,
      localConnect: null,
      remoteConnect: null,
      localStream: null,
      remoteStream: null,
      room_id: null,
      configuration: {
        iceServers: [
          {
            url: "stun:stun.l.google.com:19302"
          }
        ]
      }
    };
  },
  methods: {
    send(message) {
      this.ws.send(JSON.stringify(message));
    },
    createRoom() {
      this.send({ create_room: {} });
    },
    joinRoom() {
      this.send({ join_room: { room_id: this.room_id } });
    },
    roomSend(message) {
      message.room_id = this.room_id;
      this.send({ room_send_data: message });
    },
    onCandidate(data) {
      this.localConnect.addIceCandidate(new RTCIceCandidate(data));
    },
    onAnswer(data) {
      this.localConnect.setRemoteDescription(new RTCSessionDescription(data));
    },
    onOffer(data) {
      console.log("on offer .........");
      console.log(data);
      this.localConnect.setRemoteDescription(new RTCSessionDescription(data));
      console.log("offer.......");
      this.localConnect.createAnswer(this.createAnswer, function(err) {
        console.log(err);
      });
    },
    onMessage(message) {
      console.log(message);
      var res = JSON.parse(message.data);
      for (var key in res) {
        switch (key) {
          case "on_room_message":
            var data = JSON.parse(res[key].data);
            switch (res[key].type_name) {
              case "offer":
                this.onOffer(data);
                break;
              case "answer":
                this.onAnswer(data);
                break;
              case "candidate":
                this.onCandidate(data);
                break;
            }
            break;
          case "on_create_room":
            this.room_id = res[key].id;
            console.log(this.room_id);
            break;
          case "on_join_room":
            console.log(res);
            break;
        }
      }
    },
    onAddstream(stream) {
      console.log("add remote");
      var remoteVideo = document.querySelector("#remoteVideo");
      this.remoteStream = stream.stream;
      remoteVideo.srcObject = stream.stream;
    },
    createOffer(offer) {
      console.log(offer);
      this.roomSend({ type: "offer", data: offer });
      this.localConnect.setLocalDescription(offer);
      console.log("create offer end.");
    },
    start() {
      this.localConnect.createOffer(this.createOffer, function(error) {
        alert("An error has occurred.");
      });
    },
    createAnswer(answer) {
      this.localConnect.setLocalDescription(answer);
      this.roomSend({ type: "answer", data: answer });
    },
    onIceCandidate(event) {
      if (event.candidate) {
        console.log(event);
        this.roomSend({ type: "candidate", data: event.candidate });
      }
    },
    onLocalStream(mystream) {
      var localVideo = document.querySelector("#localVideo");
      this.localStream = mystream;
      localVideo.srcObject = mystream;
      console.log(mystream);
      if (this.localConnect == null) {
        this.localConnect = new RTCPeerConnection(this.configuration);
        this.localConnect.addStream(mystream);
        this.localConnect.onaddstream = this.onAddstream;
        this.localConnect.onicecandidate = this.onIceCandidate;
      }
    },
    link() {
      navigator.getUserMedia(
        {
          video: true,
          audio: true
        },
        this.onLocalStream,
        function(error) {
          console.log(error);
        }
      );
    }
  },
  mounted() {
    this.ws = new WebSocket("ws://localhost:8083/ws/bbb");
    this.ws.onopen = function() {
      console.log("Connected");
    };
    this.ws.onmessage = this.onMessage;
    this.ws.onclose = function() {
      console.log("Closed.");
    };
    this.link();
  }
};
</script>
