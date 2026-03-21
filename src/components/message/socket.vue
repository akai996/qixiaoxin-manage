<template>
    <div>
        <audio id="chatAudio">
        <source src="../../assets/voice/notify.ogg" type="audio/ogg">
        <source src="../../assets/voice/notify.mp3" type="audio/mpeg">
        <source src="../../assets/voice/notify.wav" type="audio/wav">
        </audio>
    </div>
</template>

<script>
    import Vue from 'vue'
    import Lockr from 'lockr'
    import CryptoJS from 'crypto-js'

    const MAX_RECONNECT = 8;          // 最大重连次数
    const HEARTBEAT_INTERVAL = 25000; // 心跳间隔(ms)
    const BASE_RECONNECT_DELAY = 2000;  // 基础重连延迟(ms)
    const MAX_RECONNECT_DELAY = 30000;  // 最大重连延迟(ms)

    export default {
        name: "socket",
        props:{
            isQrLogin:{
                type:Boolean,
                default:false
            }
        },
        data() {
            return {
                websocket: null,
                is_open_socket: false,
                connectNum: 1,
                manualClose: false,       // 是否主动关闭
                openTime: null,           // 连接打开时间
                missedPongs: 0,           // 未收到 pong 的次数
                heartbeatInterval: null,
                reconnectTimeOut: null,
            }
        },
        methods: {
            // 生成签名（与 request.js 保持一致）
            generateSign(token) {
                const timestamp = Math.floor(Date.now() / 1000).toString();
                if (!token) {
                    return { timestamp, sign: '' };
                }
                const sign = CryptoJS.HmacSHA256(timestamp, token).toString();
                return { timestamp, sign };
            },
            // 获取 WebSocket 地址
            getWsUrl(){
                let ws_host = process.env.VUE_APP_BASE_API;
                let protocol = window.location.protocol;
                let wsProtocol = "ws://";
                if(process.env.NODE_ENV === 'production'){
                    ws_host = window.location.host;
                }
                if(protocol === "https:"){
                    wsProtocol = "wss://";
                }
                return wsProtocol + ws_host + "/wss";
            },
            // 关闭旧连接，防止多个 WebSocket 同时存活
            _closeExisting() {
                if (this.websocket) {
                    try {
                        this.websocket.onclose = null; // 防止触发重连
                        this.websocket.close();
                    } catch (e) {
                        // 旧连接可能已断开，忽略
                    }
                    this.websocket = null;
                }
            },
            // 初始化 WebSocket
            initWebSocket() {
                this._closeExisting();
                this.manualClose = false;
                const WS_URI = this.getWsUrl();
                this.websocket = new WebSocket(WS_URI);
                this.is_open_socket = true;
                this.$store.state.wsStatus = true;

                this.websocket.onopen = () => {
                    this.openTime = Date.now();
                    this.connectNum = 1;
                    this.missedPongs = 0;
                    clearTimeout(this.reconnectTimeOut);
                    clearTimeout(this.heartbeatInterval);
                    this.start();
                    console.log("WebSocket 连接成功");
                };

                this.websocket.onmessage = this.websocketOnMessage;
                this.websocket.onclose = this.websocketClose;
                this.websocket.onerror = (e) => {
                    console.error('WebSocket 连接错误:', e);
                };

                Vue.prototype.$websocket = this.websocket;
            },
            // 消息接收
            websocketOnMessage(e) {
                const data = JSON.parse(e.data);
                let userInfo = Lockr.get('UserInfo');
                let token = Lockr.get('authToken');
                switch (data['type']) {
                    // 服务端 ping 客户端
                    case 'ping':
                        this.websocketSend({type: "pong", isQrLogin: this.isQrLogin});
                        break;
                    // 客户端收到 pong，重置心跳计数
                    case 'pong':
                        this.missedPongs = 0;
                        break;
                    case 'codeLoginQr':
                        this.$store.state.qrLoginUrl = data['qrurl'];
                        break;
                    case 'tokenLogin':
                        let info = data['data'];
                        this.$store.commit('SET_AUTH', info);
                        this.$store.commit('SET_USERINFO', info.userInfo);
                        setTimeout(() => {
                            window.location.reload();
                        }, 500);
                        break;
                    // 连接初始化，绑定用户
                    case 'init':
                        Lockr.set('client_id', data['client_id']);
                        if(this.isQrLogin){
                            return;
                        }
                        this.$api.commonApi.bindClientIdAPI({
                            client_id: data['client_id'],
                            user_id: userInfo.user_id
                        }).then(res => {
                            const signData = this.generateSign(token);
                            this.websocketSend({
                                type: "bindUid",
                                user_id: userInfo.user_id,
                                token: token,
                                timestamp: signData.timestamp,
                                sign: signData.sign
                            });
                            console.log(data['client_id'], '消息服务启动成功');
                        }).catch(error => {
                            console.log('连接失败');
                        });
                        break;
                    default:
                        this.$store.commit('catchSocketAction', data);
                        break;
                }
            },
            // 连接关闭
            websocketClose(e) {
                const duration = this.openTime
                    ? ((Date.now() - this.openTime) / 1000).toFixed(1) + 's'
                    : '未知';
                console.log("WebSocket 已关闭, 持续:", duration, "code:", e?.code);
                clearTimeout(this.heartbeatInterval);
                clearTimeout(this.reconnectTimeOut);
                this.is_open_socket = false;
                this.websocket = null;

                // 主动关闭不触发重连
                if (this.manualClose) {
                    return;
                }
                // 连接持续超过 5 秒才算稳定，稳定连接断开后重置重连计数
                if (this.openTime && (Date.now() - this.openTime) > 5000) {
                    this.connectNum = 1;
                }

                if (this.connectNum < MAX_RECONNECT) {
                    this.connectNum += 1;
                    this.reconnect();
                } else {
                    this.$store.state.wsStatus = false;
                    this.connectNum = 1;
                    let userInfo = Lockr.get('UserInfo');
                    if (userInfo) {
                        this.$api.commonApi.offlineAPI({user_id: userInfo.user_id}).then(res => {
                            console.log("连接已断开 (" + (e?.code || '') + ")");
                        });
                    }
                }
            },
            // 心跳检测（使用 setTimeout 防止定时器堆积）
            start() {
                clearTimeout(this.heartbeatInterval);
                this.missedPongs = 0;
                const heartbeat = () => {
                    if (this.is_open_socket && this.websocket && this.websocket.readyState === 1) {
                        this.missedPongs++;
                        // 连续 2 次心跳无 pong 响应，判定为僵尸连接，主动重连
                        if (this.missedPongs >= 2) {
                            console.warn('[WS] 心跳无响应，主动重连');
                            this.is_open_socket = false;
                            this.websocket = null;
                            this.reconnect();
                            return;
                        }
                        this.websocketSend({"type": "ping"});
                    }
                    this.heartbeatInterval = setTimeout(heartbeat, HEARTBEAT_INTERVAL);
                };
                this.heartbeatInterval = setTimeout(heartbeat, HEARTBEAT_INTERVAL);
            },
            // 发送消息
            websocketSend(agentData) {
                if (this.websocket && this.websocket.readyState === 1) {
                    this.websocket.send(JSON.stringify(agentData));
                }
            },
            // 检测连接状态
            checkStatus(){
                if(!this.websocket || [2, 3].includes(this.websocket.readyState)){
                    console.log("未连接！");
                    return false;
                }
                return true;
            },
            // 主动关闭
            close() {
                clearTimeout(this.heartbeatInterval);
                clearTimeout(this.reconnectTimeOut);
                this.manualClose = true;
                if (!this.is_open_socket) {
                    return;
                }
                if (this.websocket) {
                    this.websocket.close();
                }
            },
            // 重连（指数退避：2s, 3s, 4.5s, 6.75s, ... 最大 30s）
            reconnect(){
                clearTimeout(this.heartbeatInterval);
                clearTimeout(this.reconnectTimeOut);
                if (!this.is_open_socket && !this.manualClose) {
                    const delay = Math.min(
                        BASE_RECONNECT_DELAY * Math.pow(1.5, this.connectNum - 1),
                        MAX_RECONNECT_DELAY
                    );
                    console.log(Math.round(delay / 1000) + "秒后重新连接...(第" + this.connectNum + "次)");
                    this.reconnectTimeOut = setTimeout(() => {
                        this.initWebSocket();
                    }, delay);
                }
            },
            playAudio () {
                const audio = document.getElementById('chatAudio');
                audio.currentTime = 0;
                audio.play();
            }
        },
        created() {
            this.initWebSocket();
        },
        beforeDestroy() {
            this.close();
            clearTimeout(this.heartbeatInterval);
            clearTimeout(this.reconnectTimeOut);
        }
    }
</script>
