<template>
    <div class="chatroom style-2">
        <template v-for="(chat) in sortedChats">
            <div v-if="chat.takeBack"
                 :key="chat.uuid"
                 class="msg-wrap justify-center"
            >
                <div class="take-back-msg">
                    <span> {{chat.name}} 已收回訊息 </span>
                </div>
            </div>
            <div v-else
                 :key="chat.uuid"
                 class="msg-wrap"
                 :class="[chat.team === 'right' && 'row-reverse']"
            >
                <div class="avatar-wrap">
                    <img class="head-img" :src="imgSrc(chat.avatar)" :alt="chat.avatar">
                    <span class="name-text">{{chat.name}}</span>
                </div>
                <div>
                    <div class="msg" :class="[chat.team]" @contextmenu="openMenu(chat)">
                        <img v-if="chat.type === 'image'" width="100%" :src="chat.base64" :alt="chat.avatar">
                        <span v-else class="break-words">{{chat.msg}}</span>
                    </div>
                </div>
            </div>
        </template>
        <div class="input-wrap">
            <div class="input-left">
                <i class="flaticon flaticon-happy cursor-pointer" title="emoji"></i>
                <i class="flaticon flaticon-image cursor-pointer" title="圖片" @click="uploadImage"></i>
                <i class="flaticon flaticon-attachment cursor-pointer" title="檔案"></i>
            </div>
            <input ref="text-input" class="input" v-model="text" @keyup.enter="submit" placeholder="輸入文字"/>
            <div class="input-right" @click="submit">
                <i class="flaticon flaticon-send-button cursor-pointer" title="送出"></i>
            </div>
        </div>
    </div>
</template>

<script>
    import firebase from 'firebase';
    import firestoreUtils from '../firestore/firestoreUtils';

    export default {
        name: "Chatroom",
        created() {

            // 複製訊息
            window.ipcRenderer.on('chatroom:copy-msg', (event, chat) => this.copyMessage(chat));

            //  刪除訊息
            window.ipcRenderer.on('chatroom:delete-msg', (event, chat) => this.deleteMessage(chat));

            //  收回訊息
            window.ipcRenderer.on('chatroom:take-back-msg', (event, chat) => this.takeBackMessage(chat));

            //  下載圖片
            window.ipcRenderer.on('chatroom:save-img', (event, chat) => this.saveImage(chat));
        },
        mounted() {

            firebase.auth()
                .signInAnonymously()
                .then(user => {

                    // User is signed in.
                    // const isAnonymous = user.isAnonymous;
                    // const uid = user.uid;
                    window.localStorage.setItem('user', user);

                    firestoreUtils.observer
                        .observeRoom(this.roomId)
                        .on('new-message', msg => this.chats.push(msg))
                        .on('update-message', msg => {

                            const index = this.chats.findIndex(chat => chat.uuid === msg.uuid);
                            this.chats.splice(index, 1, msg);
                        })
                        .on('delete-message', msg => {

                            const index = this.chats.findIndex(chat => chat.uuid === msg.uuid);
                            this.chats.splice(index, 1);
                        });
                })
                .catch();
        },
        methods: {
            uploadImage() {

                window.ipcRenderer
                    .invoke('image:choose-image')
                    .then(({base64}) => {

                        const message = {
                            name: '你',
                            team: 'right',
                            avatar: 'cat-3.png',
                            type: 'image',
                            base64,
                            msg: '這是圖片',
                        }

                        this.addMessage(message);
                    })
                    .catch(e => console.error(e));
            },
            saveImage(chat) {

                console.log('image:save-image');

                window.ipcRenderer
                    .invoke('image:save-image', chat)
                    .then(data => console.log('data=', data))
                    .catch(e => console.error(e));
            },
            addMessage(chat) {

                firestoreUtils.sender
                    .addMessage(this.roomId, chat)
                    .catch(e => console.error(e));
            },
            copyMessage(chat) {

                this.text = chat.msg;
                this.$refs['text-input'].focus();
            },
            deleteMessage(chat) {

                firestoreUtils.sender
                    .deleteMessage(this.roomId, chat.uuid)
                    .catch(e => console.error(e));
            },
            takeBackMessage(chat) {

                firestoreUtils.sender
                    .updateMessage(this.roomId, {uuid: chat.uuid, takeBack: true})
                    .catch(e => console.error(e));
            },
            openMenu(chat) {

                window.ipcRenderer.send('open-contextmenu', chat);
            },
            imgSrc(avatar) {

                return require(`@/assets/head/${avatar}`);
            },
            submit() {

                if (this.text) {

                    const message = {
                        name: '你',
                        team: 'right',
                        avatar: 'cat-3.png',
                        msg: this.text
                    }

                    this.addMessage(message);

                    this.text = '';
                }
            }
        },
        computed: {
            sortedChats() {

                return [...this.chats].sort((a, b) => {

                    if (!a.create_at) return 1;
                    else if (!b.create_at) return -1;
                    else if (!b.create_at) return -1;
                    else return a.create_at.seconds - b.create_at.seconds;
                });
            }
        },
        data() {

            return {
                roomId: 'init-room',
                text: "",
                chats: []
            }
        }
    }
</script>

<style scoped>

    .input-wrap {
        user-select: none;
        width: 70vw;
        padding: 10px;
        display: flex;
        align-items: center;
        position: fixed;
        bottom: 20px;
        left: 50vw;
        transform: translateX(-50%);
        background-color: #b1a6a6;
    }

    .input-left {
        flex: 2;
        margin-right: 10px;
        height: 100%;
        display: flex;
        justify-content: space-evenly;
        align-items: center;
    }

    .input-right {
        flex: 1;
        display: flex;
        justify-content: center;
        align-items: center;
    }

    .input {
        flex: 10;
        background-color: #b1a6a6;
        border-right-style: solid;
        border-left-style: solid;
        border-top-style: unset;
        border-bottom-style: unset;
        color: white;
        padding-left: 10px;
        font-size: 22px;
    }

    .chatroom {
        overflow-y: auto;
        padding-bottom: 30px;
        min-height: calc(100vh - var(--title-bar-height));
        max-height: calc(100vh - var(--title-bar-height));
        display: flex;
        flex-direction: column;
        background-color: var(--bg-color);
    }

    .avatar-wrap {
        display: flex;
        flex-direction: column;
        align-items: center;
    }

    .msg-wrap {
        display: flex;
        margin-top: 10px;
    }

    .justify-center {
        justify-content: center;
    }

    .row-reverse {
        flex-direction: row-reverse;
    }

    .break-words {
        word-wrap: break-word;
    }

    .head-img {
        margin: 12px 16px 5px 16px;
        width: 50px;
        height: 50px;
    }

    .name-text {
        margin-left: 16px;
        margin-right: 16px;
        color: white;
    }

    .take-back-msg {
        padding: 3px 20px 3px 20px;
        border-radius: 30px;
        color: white;
        background-color: var(--take-back-msg-color);
    }

    .msg {
        color: white;
        max-width: 50vw;
        padding: 12px;
        margin: 8px 0 0 0;
        position: relative;
        border-radius: 5px;
        background-color: var(--msg-color);
    }

    /* css 三角形 : https://www.footmark.info/web-design/css/css-border-create-triangle/ */
    .msg.left::before {
        position: absolute;
        top: 15px;
        left: -27px;
        content: "";
        border-color: transparent var(--msg-color) transparent transparent;
        border-style: solid solid solid solid;
        border-width: 7px 15px 7px 15px;

        /* 設定 width、height 可更好理解原理 */
        height: 0;
        width: 0;
    }

    .msg.right::before {
        position: absolute;
        top: 15px;
        right: -27px;
        content: "";
        border-color: transparent transparent transparent var(--msg-color);
        border-style: solid solid solid solid;
        border-width: 7px 15px 7px 15px;

        /* 設定 width、height 可更好理解原理 */
        height: 0;
        width: 0;
    }

</style>