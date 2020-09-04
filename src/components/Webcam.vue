<template>
    <div class="webcam">
        <div class="sending-webcam">
            <h1>Sending Webcam</h1>
            <video id="sending-cam" autoplay="true"></video>
            <button v-on:click="startWebcam()">Start</button>
            <button v-on:click="getConnectedUsers()">GetUsers</button>
            <select v-model="selected_user" v-on:change="call(selected_user)" class="form-control" id="users-list">
                <option v-for="(user, index) in connected_users" :value="user"
                        v-bind:key="index">{{ user }}</option>
            </select>
        </div>
        <div class="receiving-webcam">
            <h1>Receiving Webcam</h1>
            <video id="receiving-cam" autoplay="true"></video>
        </div>
    </div>
</template>

<script>
export default {
    name: 'Webcam',
    data: function() {
        return {
            connection: null,
            selectedSource: null,
            sources: null,
            stream: null,
            peerConnection: null,
            connected_users: null,
            selected_user: null,
            config: {
                iceServers: [
                    {
                        urls: ["stun:stun.l.google.com:19302"]
                    }
                ]
            },
            constraints: {
                audio: true,
                video: true,
            },
        }
    },
    mounted: function() {
        const supported = 'mediaDevices' in navigator;

        if (!supported) {
            console.log("Camera cannot be accessed");
            return
        }

        this.getDevices().then(this.gotDevices).catch(() => {console.log("Error with get devices.")})
    },
    methods: {
        call: function(user) {
            if (user) {
                this.sendMessage("call", user)
            }
        },
        getDevices: async function() {
            return navigator.mediaDevices.enumerateDevices();
        },
        gotDevices: function(devices) {
            this.sources = [];
            devices.forEach(device => {
                if (device.kind === 'videoinput') {
                    this.sources.push(device)
                }
            });

            if (!this.sources.length) {
                throw 'There are no video sources found!'
            }

            if (!this.selectedSource) {
                this.selectedSource = this.sources[0]
            }
        },
        getConnectedUsers: function() {
            if (!this.connection) {
                console.log("Please start streaming first (to be changed)")
                return
            }

            this.sendMessage("get_connected_users")
        },
        startWebcam: async function() {
            if (this.connection) {
                console.log("Webcam already started!");
                return
            }

            try {
                this.stream = await navigator.mediaDevices.getUserMedia(this.constraints);
                const video = document.querySelector("#sending-cam");
                video.srcObject = this.stream;
            } catch(err) {
                console.log("Something went wrong: ", err);
            }

            console.log("Starting Connection to Websocket Server");

            this.connection = new WebSocket("wss://go-signaling.herokuapp.com/echo");

            this.connection.onopen = (event) => {
                console.log(event);
                this.sendMessage("connect");
                console.log("Successfully connected to the echo websocket server!");
            }

            this.connection.onmessage = (event) => {
                console.log(event.data)
                let data = JSON.parse(event.data);
                switch(data.message) {
                    case "users_list":
                        console.log(data.data);
                        this.connected_users = data.data;
                        break;
                    case "ready_to_establish_connection":
                        this.peerConnection = new RTCPeerConnection(this.config);
                        this.stream.getTracks().forEach(track => this.peerConnection.addTrack(track, this.stream));
                        this.peerConnection.onicecandidate = event => {
                            if (event.candidate) {
                                this.sendMessage("candidate", event.candidate);
                            }
                        }
                        this.peerConnection.createOffer()
                        .then(sdp => this.peerConnection.setLocalDescription(sdp))
                        .then(() => {
                            this.sendMessage("offer", this.peerConnection.localDescription);
                        });
                        break;
                    case "offer":
                        console.log(data.data);
                        this.peerConnection = new RTCPeerConnection(this.config);
                        this.stream.getTracks().forEach(track => this.peerConnection.addTrack(track, this.stream));
                        this.peerConnection.setRemoteDescription(data.data)
                        .then(() => this.peerConnection.createAnswer())
                        .then(sdp => this.peerConnection.setLocalDescription(sdp))
                        .then(() => {
                            this.sendMessage("answer", this.peerConnection.localDescription);
                        });
                        this.peerConnection.ontrack = event => {
                            const video = document.querySelector("#receiving-cam");
                            video.srcObject = event.streams[0];
                        }
                        this.peerConnection.onicecandidate = event => {
                            if (event.candidate) {
                                this.sendMessage("candidate", event.candidate);
                            }
                        }
                        break;
                    case "answer":
                        this.peerConnection.setRemoteDescription(data.data);
                        this.peerConnection.ontrack = event => {
                            console.log("here!!!");
                            console.log(event);
                            const video = document.querySelector("#receiving-cam");
                            video.srcObject = event.streams[0];
                        }
                        break;
                    case "candidate":
                        this.peerConnection.addIceCandidate(new RTCIceCandidate(data.data));
                        break;
                }
            }

            this.connection.onclose = (event) => {
                console.log(event);
                this.connection = null;
            }
        },
        sendMessage: function(message_type, data=null) {
            let message = {
                message: message_type,
                data: data,
            }

            this.connection.send(JSON.stringify(message));
        },
    },
}
</script>

<style scoped>
video {
    background: black;
    border: 1px solid black;
    border-radius: 400px;
    width: 400px;
    height: 400px;
    margin: 10px;
}

.webcam {
    display: flex;
    flex-direction: row;
    justify-content: space-around;
}

.sending-webcam {
    display: flex;
    flex-direction: column;
}
</style>