<template>
  <div>
    <div class="container text-center" v-show="show">
      <div class="row">
        <div class="col-md-4 col-md-offset-4">
          <form class="form" action @submit.prevent="submit()">
            <h2>WebRTC Demmo Project3 Mời Bạn Đăng Nhập</span></h2>
            <br />
            <input
              class="form-control"
              type="text"
              placeholder="Tên Đăng Nhập"
              required
              autofocus
              v-model="user_name"
            />
            <br />
            <button class="btn btn-primary btn-block" type="submit">Đăng Nhập</button>
          </form>
        </div>
      </div>
    </div>
    <div class="container text-center" v-show="!show">
      <div class="row">
        <div class="col-md-3" style="height: 50%">
          <ul class="list-group">
            <li class="list-group-item">Tên đăng nhập: {{ user_name }}</li>
            <li class="list-group-item">Số người đang online: {{ Object.getOwnPropertyNames(users).length - 1 }}</li>
            <li class="list-group-item">
              Người dùng:
              <div v-for="(user, index) in users" :key="index">
                <br />
                <span>{{ index }}</span>
                <span :class="[user ? 'green_color' : 'red_color']">{{
                  user ? '(Trực tuyến)' : '(Đang có cuộc gọi))'
                }}</span>
              </div>
            </li>
          </ul>

          <div class="row text-center">
            <div class="col-md-12">
              <input class="form-control" type="text" v-model="call_username" placeholder="username to call" />
              <br />
              <button class="btn-success btn" :disabled="!users[user_name]" @click="call">Call</button>
              <button class="btn-danger btn" :disabled="users[user_name]" @click="hangUp">Hang Up</button>
            </div>
          </div>
          <button
            onclick="window.location.href='http://18.216.47.0:5000/?room=thangdp_412418611'"
            class="btn-success btn"
            style="margin: 30px"
          >
            Gọi Nhóm
          </button>
        </div>
        <div class="col-md-9" style="flex-direction: row">
          <video id="localVideo" autoplay></video>
          <video
            id="remoteVideo"
            src=""
            autoplay="autoplay"
            style="/*! height: 100-x; */ height: 100px; width: 100px"
          ></video>
        </div>
      </div>
    </div>
    <div class="preview" v-show="accept_video">
      <div class="preview-wrapper">
        <div class="preview-container">
          <div class="preview-body">
            <h4>Bạn có cuộc gọi video Bạn chấp nhận không?</h4>
            <button class="btn-success btn" @click="accept">Đồng Ý</button>
            <button class="btn-danger btn" style="margin-right: 70px" @click="reject">Từ Chối</button>
          </div>
          <div class="confirm" @click="closePreview">×</div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import * as config from '../../configure';

navigator.getUserMedia = navigator.getUserMedia || navigator.mozGetUserMedia || navigator.webkitGetUserMedia;
window.RTCPeerConnection = window.RTCPeerConnection || window.mozRTCPeerConnection || window.webkitRTCPeerConnection;
window.RTCIceCandidate = window.RTCIceCandidate || window.mozRTCIceCandidate || window.webkitRTCIceCandidate;
window.RTCSessionDescription =
  window.RTCSessionDescription || window.mozRTCSessionDescription || window.webkitRTCSessionDescription;

const socket = io.connect(config.API_ROOT);
const configuration = {
  iceServers: [config.DEFAULT_ICE_SERVER],
};

let localStream, peerConn;
let connectedUser = null;

export default {
  data() {
    return {
      user_name: '',
      show: true,
      users: '',
      call_username: '',
      remote_video: '',
      accept_video: false,
    };
  },
  mounted() {
    socket.on(
      'message',
      function (data) {
        console.log(data);
        switch (data.event) {
          case 'show':
            this.users = data.allUsers;
            break;
          case 'join':
            this.handleLogin(data);
            break;
          case 'call':
            this.handleCall(data);
            break;
          case 'accept':
            this.handleAccept(data);
            break;
          case 'offer':
            this.handleOffer(data);
            break;
          case 'candidate':
            this.handleCandidate(data);
            break;
          case 'msg':
            this.handleMsg(data);
            break;
          case 'answer':
            this.handleAnswer(data);
            break;
          case 'leave':
            this.handleLeave();
            break;
          default:
            break;
        }
      }.bind(this)
    );
  },
  methods: {
    submit() {
      if (this.user_name !== '') {
        this.send({
          event: 'join',
          name: this.user_name,
        });
      }
    },
    send(message) {
      if (connectedUser !== null) {
        message.connectedUser = connectedUser;
      }
      socket.send(JSON.stringify(message));
    },
    handleLogin(data) {
      if (data.success === false) {
        alert('Tên đã trùng mời chọn trên khác');
      } else {
        this.show = false;
        this.users = data.allUsers;
        this.initCreate();
      }
    },
    addVideoURL(elementId, stream) {
      var video = document.getElementById(elementId);
      // Older brower may have no srcObject
      if ('srcObject' in video) {
        video.srcObject = stream;
      } else {
        video.src = window.URL.createObjectURL(stream);
      }
    },
    initCreate() {
      const self = this;
      navigator.mediaDevices
        .getUserMedia({ audio: true, video: true })
        .then(function (stream) {
          var video = document.getElementById('localVideo');
          self.addVideoURL('localVideo', stream);
          video.muted = true;
          localStream = stream;
        })
        .catch(function (err) {
          console.log(err.name + ': ' + err.message);
        });
    },
    call() {
      if (this.call_username.length > 0) {
        if (this.users[this.call_username] === true) {
          connectedUser = this.call_username;
          this.createConnection();
          this.send({
            event: 'call',
          });
        } else {
          alert('Người dùng bận');
        }
      } else {
        alert('Vui lòng thử lại');
      }
    },
    createConnection() {
      peerConn = new RTCPeerConnection(configuration);
      peerConn.addStream(localStream);
      peerConn.onaddstream = (e) => {
        this.addVideoURL('remoteVideo', e.stream);
      };
      peerConn.onicecandidate = (event) => {
        setTimeout(() => {
          if (event.candidate) {
            this.send({
              event: 'candidate',
              candidate: event.candidate,
            });
          }
        });
      };
    },
    handleCall(data) {
      this.accept_video = true;
      connectedUser = data.name;
    },
    reject() {
      this.send({
        event: 'accept',
        accept: false,
      });
      this.accept_video = false;
    },
    accept() {
      this.send({
        event: 'accept',
        accept: true,
      });
      this.accept_video = false;
    },
    handleAccept(data) {
      if (data.accept) {
        // Create an offer
        peerConn.createOffer(
          (offer) => {
            this.send({
              event: 'offer',
              offer: offer,
            });
            peerConn.setLocalDescription(offer);
          },
          (error) => {
            alert('Error when creating an offer');
          }
        );
      } else {
        alert('对方已拒绝');
      }
    },
    handleOffer(data) {
      connectedUser = data.name;
      this.createConnection();
      peerConn.setRemoteDescription(new RTCSessionDescription(data.offer));
      // Create an answer to an offer
      peerConn.createAnswer(
        (answer) => {
          peerConn.setLocalDescription(answer);
          this.send({
            event: 'answer',
            answer: answer,
          });
        },
        (error) => {
          alert('Error when creating an answer');
        }
      );
    },
    handleMsg(data) {
      console.log(data.message);
    },
    handleAnswer(data) {
      peerConn.setRemoteDescription(new RTCSessionDescription(data.answer));
    },
    handleCandidate(data) {
      peerConn.addIceCandidate(new RTCIceCandidate(data.candidate));
    },
    hangUp() {
      this.send({
        event: 'leave',
      });
      this.handleLeave();
    },
    handleLeave() {
      alert('Cuộc gọi kết thúc');
      connectedUser = null;
      this.remote_video = '';
      peerConn.close();
      peerConn.onicecandidate = null;
      peerConn.onaddstream = null;
      if (peerConn.signalingState === 'closed') {
        this.initCreate();
      }
    },
    closePreview() {
      this.accept_video = false;
    },
  },
};
</script>

<style>
.preview {
  position: fixed;
  z-index: 9998;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.5);
  display: table;
  transition: opacity 0.3s ease;
}
.preview-wrapper {
  display: table-cell;
  vertical-align: middle;
}
.preview-container {
  width: 400px;
  height: 150px;
  margin: 0px auto;
  background-color: #fff;
  border-radius: 2px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.33);
  transition: all 0.3s ease;
  font-family: Helvetica, Arial, sans-serif;
  position: relative;
}
.confirm {
  position: absolute;
  right: 10px;
  top: 0px;
  font-size: 40px;
}
.confirm:hover {
  color: red;
  cursor: pointer;
}
.preview-body {
  position: absolute;
  width: 380px;
  height: 130px;
  margin: 10px 10px 10px 10px;
}
.preview-body > h4 {
  position: absolute;
  top: 25%;
  left: 20%;
}
.preview-body > button {
  position: absolute;
  right: 10px;
  bottom: 0px;
}
.green_color {
  color: green;
}
.red_color {
  color: red;
}
</style>
