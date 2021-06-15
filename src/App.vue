<template>
    <div class="h-screen w-full flex bg-gray-800 text-white">
        <div class="w-96 p-4 flex">
            <div
                class="flex-1 bg-gray-700 rounded-lg  p-4 shadow-2xl flex flex-col"
            >
                <div>
                    <div class="flex items-center mb-3">
                        <h1 class="">
                            My Id: <span class="font-bold">{{ myId }}</span>
                        </h1>
                        <v-button @click="toggleVideoShow"
                            >{{
                                isVideoShowing ? "Hide" : "Show"
                            }}
                            Video</v-button
                        >
                    </div>

                    <div class="flex">
                        <v-input
                            v-model="connectTo"
                            placeholder="Receiver Id"
                        ></v-input>
                        <v-button @click="connectMe">Connect</v-button>
                    </div>

                    <div class="my-2" v-if="loading">
                        <v-rotate-loader></v-rotate-loader>
                    </div>

                    <h1 v-if="error" class="mx-2">{{ error }}</h1>

                    <h1 v-for="client in connectedClients" :key="client">
                        {{ client }} Connected
                    </h1>
                </div>

                <div class="flex-1 overflow-y-scroll my-2">
                    <h1 v-for="message in messages" :key="message.id">
                        {{ message.message }}
                    </h1>
                </div>
                <form @submit.prevent="sendMessage" class="flex">
                    <v-input placeholder="Message" v-model="message"></v-input>
                    <v-button type="submit">Send</v-button>
                </form>
            </div>
        </div>
        <div class="flex-1 flex p-4">
            <div
                class="flex-1 bg-gray-700 rounded-md shadow-2xl flex p-4 flex-wrap overflow-y-scroll justify-center items-center"
            >
                <transition name="video">
                    <video
                        v-if="isVideoShowing"
                        ref="myVideo"
                        class="w-96 h-96 object-cover bg-gray-700 m-2  border-gray-900 rounded-lg border-4 shadow-2xl"
                        autoplay
                    />
                </transition>

                <transition-group name="video">
                    <video
                        v-for="(value, name) in videoStreams"
                        :key="name"
                        :ref="createRef(name)"
                        class="w-96 h-96 object-cover bg-gray-700 m-2  border-gray-900 rounded-lg border-4 shadow-2xl"
                        autoplay
                    />
                </transition-group>
            </div>
        </div>
    </div>
</template>

<script>
const types = {
    CONNECTED_CLIENTS: "CONNECTED_CLIENTS",
    MESSAGE: "MESSAGE",
};
import Peer from "peerjs";
import VButton from "./components/VButton.vue";
import VInput from "./components/VInput.vue";
import VRotateLoader from "./components/VRotateLoader.vue";

export default {
    components: { VButton, VInput, VRotateLoader },
    name: "App",
    data() {
        return {
            loading: false,
            error: "",
            isVideoShowing: true,
            myId: "",
            message: "",
            connectTo: "",
            conn: null,
            peer: null,
            myStream: null,
            messages: [],
            remoteStreams: {
                // peerid: {
                //   connection: connectionObj,
                //   call: callObject,
                // }
            },
            connectedClients: [],
            videoStreams: {},
        };
    },
    computed: {},
    watch: {
        connectedClients(value) {
            console.log(value);
        },
    },
    methods: {
        toggleVideoShow() {
            this.isVideoShowing = !this.isVideoShowing;
            if (this.isVideoShowing) {
                this.$nextTick(() => {
                    this.$refs.myVideo.srcObject = this.myStream;
                });
            }
        },
        createRef(id) {
            return "video-" + id;
        },
        createMessage(message) {
            return {
                id: new Date().toISOString(),
                message: message,
            };
        },
        sendMessage() {
            const message = this.createMessage(this.message);
            for (const peerId in this.remoteStreams) {
                this.remoteStreams[peerId].connection.send({
                    type: types.MESSAGE,
                    data: message,
                });
            }
            this.messages.push(message);
        },

        [types.CONNECTED_CLIENTS]: function(connectedClients) {
            for (const callPeerId of connectedClients) {
                const connObj = this.peer.connect(callPeerId);
                const callObj = this.peer.call(callPeerId, this.myStream);
                const peerId = connObj.peer;
                this.remoteStreams[peerId] = {
                    connection: connObj,
                    call: callObj,
                };

                const connectedPeer = this.remoteStreams[peerId];

                connectedPeer.connection.on("open", () => {
                    this.connectedClients.push(peerId);
                    connectedPeer.connection.on("data", (data) =>
                        this.mapDataToType(data)
                    );
                });
                connectedPeer.call.on("stream", (stream) => {
                    this.videoStreams[peerId] = stream;
                    this.$nextTick(() => {
                        this.$refs[
                            "video-" + peerId
                        ].srcObject = this.videoStreams[peerId];
                    });
                });
            }
        },
        [types.MESSAGE]: function(message) {
            console.log(message);
            this.messages.push(message);
        },
        mapDataToType(receivedData) {
            let exist = true;
            for (const type in types) {
                if (receivedData.type === type) {
                    exist = true;
                }
            }

            if (exist) {
                this[receivedData.type](receivedData.data);
            }
        },

        initialize() {
            this.peer = new Peer();
            this.peer.on("open", (id) => {
                this.myId = id;
                console.log("My peer ID is: " + id);
            });
            // receiving end
            this.peer.on("connection", (connObj) => {
                const peerId = connObj.peer;
                this.remoteStreams[peerId] = {
                    connection: connObj,
                    call: null,
                };

                for (const peerId in this.remoteStreams) {
                    console.log(peerId, this.remoteStreams[peerId]);
                    const connectedPeer = this.remoteStreams[peerId];

                    connectedPeer.connection.on("open", () => {
                        console.log(connectedPeer, "conneceted");
                        this.connectedClients.push(peerId);
                        connectedPeer.connection.on("data", (data) =>
                            this.mapDataToType(data)
                        );
                    });
                }
            });
            this.peer.on("call", (call) => {
                // Answer the call, providing our mediaStream
                const peerId = call.peer;
                call.answer(this.myStream);
                this.remoteStreams[peerId].call = call;

                call.on("stream", (stream) => {
                    this.videoStreams[peerId] = stream;
                    this.$nextTick(() => {
                        this.$refs[
                            "video-" + peerId
                        ].srcObject = this.videoStreams[peerId];
                    });
                });
            });

            this.peer.on("error", () => {
                // console.log(err);
                this.error = "Could not connect with given id";
                this.loading = false;
            });

            this.peer.on("disconnect", () => {
                console.log(this.peer.id, " disconnected");
            });
        },
        connectMe() {
            const connObj = this.peer.connect(this.connectTo);
            const callObj = this.peer.call(this.connectTo, this.myStream);
            this.loading = true;
            const peerId = connObj.peer;
            this.remoteStreams[peerId] = { connection: connObj, call: callObj };
            this.connectTo = "";

            for (const peerId in this.remoteStreams) {
                console.log(peerId, this.remoteStreams[peerId]);
                const connectedPeer = this.remoteStreams[peerId];

                connectedPeer.connection.on("open", () => {
                    // when i am calling someone
                    this.loading = false;
                    console.log(connectedPeer, "conneceted");
                    connectedPeer.connection.send({
                        type: types.CONNECTED_CLIENTS,
                        data: this.connectedClients,
                    });
                    this.connectedClients.push(peerId);
                    connectedPeer.connection.on("data", (data) =>
                        this.mapDataToType(data)
                    );
                });

                connectedPeer.call.on("stream", (stream) => {
                    this.loading = false;
                    this.videoStreams[peerId] = stream;
                    this.$nextTick(() => {
                        this.$refs[
                            "video-" + peerId
                        ].srcObject = this.videoStreams[peerId];
                    });
                });
            }
        },
    },
    created() {
        this.initialize();
        navigator.mediaDevices
            .getUserMedia({ video: true, audio: false })
            .then((stream) => {
                this.myStream = stream;
                this.$refs.myVideo.srcObject = stream;
            })
            .catch((err) => {
                console.error("Failed to get local stream", err);
            });
    },
};
</script>

<style>
#app {
    font-family: Avenir, Helvetica, Arial, sans-serif;
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
}

.video-enter-from {
    -webkit-transform: scale(2);
    transform: scale(2);
    -webkit-filter: blur(4px);
    filter: blur(4px);
    opacity: 0;
}

.video-enter-to {
    -webkit-transform: scale(1);
    transform: scale(1);
    -webkit-filter: blur(0px);
    filter: blur(0px);
    opacity: 1;
}

.video-leave-from {
    opacity: 1;
}

.video-leave-to {
    opacity: 0;
}

.video-enter-active,
.video-leave-active {
    transition: transform, filter,
        opacity 0.7s cubic-bezier(0.47, 0, 0.745, 0.715);
}
</style>
