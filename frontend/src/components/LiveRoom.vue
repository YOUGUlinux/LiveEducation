<template>
    <div id="bg">
        <div id="cover"></div>
        <div id="live-room">
            <div class="header">
                <home-page-header></home-page-header>
            </div>
            <div class="navigation">
                <div class="navigation-content">
                    <div class="welcome">
                        <Icon type="university"></Icon>
                        <label>欢迎进入直播间 !</label>
                    </div>
                    <div class="navigation-center">
                        <label class="information">老师姓名：{{ this.teacherName }}</label>
                        <label class="information">房间ID:{{ this.roomId }}</label>
                        <label class="information">房间名：{{ this.roomName }}</label>
                        <label class="information">在线人数：{{ this.studentNum }}</label>
                    </div>
                    <div class="navigation-right">
                        <template v-if="this.teacherName === this.username">
                            <Button @click="startLive" type="primary" shape="circle" size="small" id="start-live-button">开始直播</Button>
                            <Button @click="closeLive" type="error" shape="circle" size="small" id="stop-live-button">结束直播</Button>
                        </template>
                    </div>
                </div>
            </div>
            <div class="layout-header">
                <div class="left-container" id="teaching-tools">
                    <teaching-tools :roomId="this.roomId" :teacherName="this.teacherName" :username="this.username" :container-height="this.teachingToolsHeight" :container-width="this.teachingToolsWidth" :tools-on-left="this.toolsOnLeft"></teaching-tools>
                </div>
                <div class="center-container">
                    <div class="buttons-panel">
                        <template v-if="this.hidden === true">
                            <Tooltip content="弹出右边窗口" placement="right-start">
                                <Button id="pop-up-button" type="ghost" @click="popUp">
                                    <Icon type="ios-redo"></Icon>
                                </Button>
                            </Tooltip>
                        </template>
                        <template v-else>
                            <Tooltip content="切换位置" placement="right-start">
                                <Button id="swap-button" type="ghost" @click="swap">
                                    <Icon type="arrow-swap"></Icon>
                                </Button>
                            </Tooltip><br>
                            <Tooltip content="隐藏右边窗口" placement="right-start">
                                <Button id="hide-button" type="ghost" @click="hideRight">
                                    <Icon type="ios-undo"></Icon>
                                </Button>
                            </Tooltip><br>
                            <Tooltip content="隐藏左边窗口" placement="right-start">
                                <Button id="hide-button" type="ghost" @click="hideLeft">
                                    <Icon type="ios-redo"></Icon>
                                </Button>
                            </Tooltip>
                        </template>
                    </div>
                </div>
                <div class="right-up-container" id="video-display">
                    <video-display :roomId="this.roomId" :teacherName="this.teacherName" :username="this.username" :container-height="this.videoDisplayHeight"></video-display>
                </div>
                <div id="chatroom">
                    <chat-board v-on:stuNum="getNum" :roomId="this.roomId" :teacherName="this.teacherName" :username="this.username" :container-height="this.chatBoardHeight"></chat-board>
                </div>
            </div>
            <div>
                <page-footer></page-footer>
            </div>
        </div>
    </div>
</template>

<script src="/socket.io/socket.io.js"></script>
<script>
/**
 * 直播间页面，顶部包含导航栏和房间信息，
 * 页面主体部分包括教学区域、视频区域和聊天区域，
 * 这三部分可以较为灵活的切换位置。
 * 其中，教学区域包括白板、代码编辑器、课件展示的功能，
 * 视频区域和聊天区域分别实现了视频直播和聊天功能。
 *
 * @module LiveRoom
 * @class LiveRoom
 */
import * as io from 'socket.io-client'
import myMsg from './../warning.js'
import HomePageHeader from './HomePageHeader'
import PageFooter from './PageFooter'
import VideoDisplay from './VideoDisplay'
import ChatBoard from './ChatBoard'
import TeachingTools from './TeachingTools'

export default {
    name: 'live-room',
    components: {
        HomePageHeader,
        PageFooter,
        ChatBoard,
        VideoDisplay,
        TeachingTools
    },
    data: function () {
        return {
            /**
             * 房间ID
             *
             * @attribute roomId
             * @type Number
             * @default -1
             */
            roomId: -1,
            /**
             * 房间名称
             *
             * @attribute roomName
             * @type String
             * @default ''
             */
            roomName: '',
            /**
             * 老师名字
             *
             * @attribute teacherName
             * @type String
             * @default ''
             */
            teacherName: '',
            /**
             * 在线人数
             *
             * @attribute studentNum
             * @type String
             * @default ''
             */
            studentNum: '',
            /**
             * 用户名字
             *
             * @attribute username
             * @type String
             * @default ''
             */
            username: '',
            /**
             * 表示是否隐藏右边窗口
             *
             * @attribute hidden
             * @type Boolean
             * @default false
             */
            hidden: false,
            /**
             * 表示客户端，监听服务器传来的消息
             *
             * @attribute socket
             * @type Object
             * @default ''
             */
            socket: '',
            /**
             * 表示直播开始的时间
             *
             * @attribute startTime
             * @type String
             * @default ''
             */
            startTime: '',
            /**
             * 表示教学工具是否位于左边窗口
             *
             * @attribute toolsOnLeft
             * @type Boolean
             * @default true
             */
            toolsOnLeft: true,
            /**
             * 表示教学区域组件的高
             *
             * @attribute teachingToolsHeight
             * @type Number
             * @default 0
             */
            teachingToolsHeight: 0,
            /**
             * 表示教学区域组件的宽
             *
             * @attribute teachingToolsWidth
             * @type Number
             * @default 0
             */
            teachingToolsWidth: 0,
            /**
             * 表示视频域组件的高
             *
             * @attribute videoDisplayHeight
             * @type Number
             * @default 0
             */
            videoDisplayHeight: 0,
            /**
             * 表示聊天区域组件的高
             *
             * @attribute chatBoardHeight
             * @type Number
             * @default 0
             */
            chatBoardHeight: 0,
            /**
             * 表示
             *
             * @attribute intervalNum
             * @type Number
             * @default -1
             */
            intervalNum: -1
        }
    },
    /**
     * created函数，初始化相关数据，客户端发送'joinRoom'消息
     *
     * @method created
     */
    async created () {
        this.roomId = this.$route.params.id
        this.socket = io.connect('http://localhost:9000')
        this.socket.emit('joinRoom', this.roomId)
        let roomInfo = await this.getRoomInfo()
        if (roomInfo.result) {
            this.roomName = roomInfo.roomName
            this.studentNum = roomInfo.studentNum
            this.teacherName = roomInfo.teacherName
            this.username = roomInfo.name
        } else {
            this.$router.push({ name: 'home' })
        }
        this.intervalNum = window.setInterval(this.changeNum, 5000)
    },
    /**
     * mounted函数，初始化教学区域和白板区域的长和高，
     * 并监测窗口大小的改变，实时变化相应参数
     *
     * @method mounted
     */
    mounted: function () {
        let self = this
        this.resize(self)
        window.onresize = () => {
            let self = this
            this.resize(self)
        }
        self.socket.on('closeLive', function () {
            self.$Message.warning(myMsg.room['endLive'])
            window.clearInterval(self.intervalNum)
            if (self.teacherName === self.username) {
                setTimeout(window.close, 3000)
            } else {
                setTimeout(function () {
                    self.$router.push({ name: 'home' })
                }, 3000)
            }
        })
        self.startRecord()
    },
    methods: {
        /**
         * 当窗口大小变化时调整各个组件的大小（包括初始化）
         *
         * @method resize
         * @param self 指向this
         */
        resize: function (self) {
            document.getElementById('bg').style.height = window.innerHeight + 'px'
            document.getElementById('bg').style.width = window.innerWidth + 'px'
            self.teachingToolsWidth = document.getElementById('teaching-tools').clientWidth
            self.teachingToolsHeight = document.getElementById('teaching-tools').clientHeight
            self.videoDisplayHeight = document.getElementById('video-display').clientHeight
            if (this.hidden) {
                self.chatBoardHeight = document.getElementById('chatroom').clientHeight
            } else {
                self.chatBoardHeight = document.getElementById('chatroom').clientHeight - 12
            }
        },
        /**
         * 关闭直播
         *
         * @method closeLive
         */
        closeLive: function () {
            document.getElementById('cover').style.display = 'inline'
            this.socket.emit('closeLive', this.roomId)
            fetch('/closeLiveRoom/', {
                method: 'post',
                mode: 'cors',
                credentials: 'same-origin',
                headers: {
                    'Content-Type': 'application/json, text/plain, */*',
                    'Accept': 'application/json'
                },
                body: JSON.stringify({
                    'roomId': this.roomId
                })
            })
        },
        /**
         * 开始录播
         *
         * @method startRecord
         */
        startRecord: function () {
            let self = this
            self.socket.on('time', function (time) {
                self.startTime = time
                fetch('/startRecord/', {
                    method: 'post',
                    mode: 'cors',
                    credentials: 'same-origin',
                    headers: {
                        'Content-Type': 'application/json, text/plain, */*',
                        'Accept': 'application/json'
                    },
                    body: JSON.stringify({
                        'channel': self.roomId,
                        'time': self.startTime
                    })
                })
            })
        },
        /**
         * 开始直播，发送'startLive'消息
         *
         * @method startLive
         */
        startLive: function () {
            this.socket.emit('startLive', this.roomId)
            document.getElementById('start-live-button').style.display = 'none'
            document.getElementById('stop-live-button').style.display = 'inline-block'
        },
        /**
         * 隐藏左边窗口，将右上窗口移到左边，拉长聊天区域，
         * 调用swap、hideRight函数。
         *
         * @method hideLeft
         */
        hideLeft: function () {
            this.swap()
            this.hideRight()
        },
        /**
         * 隐藏右边窗口，同时拉长聊天区域。
         *
         * @method hideRight
         */
        hideRight: function () {
            document.getElementsByClassName('right-up-container')[0].style.display = 'none'
            this.hidden = true
            document.getElementById('chatroom').style.paddingTop = '0'
            document.getElementById('chatroom').style.height = '100%'
            document.getElementById('chatroom').style.top = 'inherit'
            this.chatBoardHeight = document.getElementById('chatroom').clientHeight
        },
        /**
         * 弹出右边窗口
         *
         * @method popUp
         */
        popUp: function () {
            document.getElementsByClassName('right-up-container')[0].style.display = 'block'
            this.hidden = false
            document.getElementById('chatroom').style.paddingTop = '12px'
            document.getElementById('chatroom').style.height = '60%'
            document.getElementById('chatroom').style.top = '40%'
            this.chatBoardHeight = document.getElementById('chatroom').clientHeight - 12
        },
        /**
         * 左右子组件交换位置
         *
         * @method swap
         */
        swap: function () {
            let teachingTools = document.getElementById('teaching-tools')
            teachingTools.className = teachingTools.className === 'left-container' ? 'right-up-container' : 'left-container'
            let videoDisplay = document.getElementById('video-display')
            videoDisplay.className = videoDisplay.className === 'left-container' ? 'right-up-container' : 'left-container'
            this.toolsOnLeft = !this.toolsOnLeft
            this.teachingToolsWidth = document.getElementById('teaching-tools').clientWidth
            this.teachingToolsHeight = document.getElementById('teaching-tools').clientHeight
            this.videoDisplayHeight = document.getElementById('video-display').clientHeight
        },
        /**
         * 更新学生数量
         *
         * @method changeNum
         */
        changeNum: function () {
            if (this.username === this.teacherName) {
                fetch('/changeNum/', {
                    method: 'post',
                    mode: 'cors',
                    credentials: 'same-origin',
                    headers: {
                        'Content-Type': 'application/json, text/plain, */*',
                        'Accept': 'applica tion/json'
                    },
                    body: JSON.stringify({
                        'studentNum': this.studentNum,
                        'roomId': this.roomId
                    })
                })
            }
        },
        /**
         * 获取学生数量
         *
         * @method getNum
         */
        getNum: function (count) {
            this.studentNum = count
        },
        /**
         * 获取房间信息
         *
         * @method getRoomInfo
         */
        getRoomInfo: function () {
            return fetch('/getRoomInfo/', {
                method: 'post',
                mode: 'cors',
                credentials: 'same-origin',
                headers: {
                    'Content-Type': 'application/json, text/plain, */*',
                    'Accept': 'application/json'
                },
                body: JSON.stringify({
                    'roomID': this.roomId
                })
            }).then((response) => response.json())
        }
    }
}
</script>

<style scoped>
#bg {
    min-height: 600px;
    min-width: 800px;
}

#live-room {
    background: transparent;
    position: relative;
    border-radius: 5px;
    overflow: hidden;
    width: 85%;
    height: 100%;
    margin-left: auto;
    margin-right: auto;
    min-height: 600px;
    min-width: 800px;
    max-width: 1200px;
}

.header {
    height: 50px;
    width: 100%;
    font-size: 40px;
    position: fixed;
    left: 0;
    z-index: 50;
}

.navigation {
    background-color: rgba(239, 239, 239, 0.6);
    padding: 8px 10px;
    overflow: hidden;
    display: flex;
    position: fixed;
    left: 0;
    top: 50px;
    width: 100%;
    font-size: 15px;
}

.navigation-content {
    width: 85%;
    min-width: 800px;
    max-width: 1200px;
    display: flex;
    margin: auto;
}

.navigation-center {
    margin: 0 auto;
    font-size: 15px;
}

.navigation-right {
    width: 79px;
    font-size: 15px;
}

#stop-live-button {
    display: none;
}

.information {
    margin: 0 auto;
}

.layout-header {
    width: 100%;
    height: 78%;
    min-width: 800px;
    display: flex;
    margin-left: auto;
    margin-right: auto;
    margin-top: 110px;
    position: relative;
}

.left-container {
    position: absolute;
    left: 0;
    height: 100%;
    width: 65%;
    text-align: left;
    overflow: hidden;
}

.center-container {
    width: 5%;
    position: absolute;
    text-align: center;
    left: 65%;
}

.right-up-container {
    display: block;
    height: 40%;
    position: absolute;
    right: 0;
    width: 30%
}

#chatroom {
    position: absolute;
    right: 0;
    top: 40%;
    padding-top: 12px;
    height: 60%;
    width: 30%;
}

#swap-button,
#pop-up-button,
#hide-button {
    height: 41px;
    border: none;
}

#cover {
    display: none;
    z-index: 999;
    left: 0;
    top: 0;
    width: 100%;
    height: 100%;
    background-color: transparent;
    position: absolute;
}
</style>
