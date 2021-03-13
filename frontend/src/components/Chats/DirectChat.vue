<template>
    <v-container class="fill-height align-start content-container" fluid>
        <v-tabs v-model="selectedFunnelIndex">
            <v-tab v-for="funnel in activeFunnels" :key="funnel.id">{{funnel.title}}</v-tab>
        </v-tabs>
        <v-row class="fill-height">
            <v-col cols="12" md="3">
                <v-list>
                    <v-subheader>Список чатов</v-subheader>
                    <v-list-item-group
                        v-model="selectedChatIndex"
                        color="primary"
                        @change="loadChatHistory"
                    >
                        <v-list-item v-for="chat in unreadChats" :key="chat.id">
                            <v-list-item-content>
                                <v-list-item-title>{{getChatTitle(chat.user)}}</v-list-item-title>
                            </v-list-item-content>
                            <v-list-item-action>
                                <v-btn icon @click="markRead(chat)"><v-icon>mdi-eye-off</v-icon></v-btn>
                            </v-list-item-action>
                        </v-list-item>
                    </v-list-item-group>
                </v-list>
            </v-col>
            <v-col cols="12" md="9" class="d-flex flex-column" v-if="selectedChat">
                <v-card v-for="message in chatMessages" :key="message.messageId"
                        class="mb-2"
                        width="70%"
                        :class="{'my align-self-end': selectedChat.id !== message.message.from.id}"
                        :color="selectedChat.id === message.message.from.id ? 'white' : 'blue'"
                        :dark="selectedChat.id !== message.message.from.id"
                >
                    <v-card-text class="pb-0 d-flex">
                        <b>{{getChatTitle(message.message.from)}}</b>
                        <small class="ml-4">{{getMessageTime(message.message)}}</small>
                        <v-spacer></v-spacer>
                        <small>{{getStageName(message)}}</small>
                    </v-card-text>

                    <v-card-text v-html="message.message.text" class="pt-0"></v-card-text>
                </v-card>
                <v-container class="footer p-0" fluid>
                    <v-sheet class="p-2">
                        <v-row>
                            <v-col cols="12">
                                <v-textarea v-model="reply[selectedChat.id+':'+selectedChat.botId]" label="Сообщение"></v-textarea>
                            </v-col>
                        </v-row>
                        <v-row>
                            <v-col cols="12">
                                <v-btn @click="sendReply">Отправить ответ <v-icon>mdi-telegram</v-icon></v-btn>
                            </v-col>
                        </v-row>
                    </v-sheet>
                </v-container>
            </v-col>
            <v-col cols="12" md="9" class="d-flex flex-column pt-4 text-center" v-else>
                Чат не выбран
            </v-col>
        </v-row>
    </v-container>
</template>

<script>
    import moment from "moment";

    export default {
        name: "DirectChat",
        async mounted() {
            await this.loadBots();
            await this.loadFunnels();
            await this.loadUnreadChats();
            this.startHistoryPolling();
        },
        beforeDestroy () {
            this.stopHistoryPolling();
        },
        data() {
            return {
                selectedChatIndex: null,
                selectedFunnelIndex: null,
                reply: {},
                pollIntervalId: false,
                pollMs: 10000,
            }
        },
        methods: {
            loadFunnels() {
                return this.$store.dispatch('funnel/loadItems');
            },
            loadBots() {
                return this.$store.dispatch('bot/loadItems');
            },
            loadUnreadChats() {
                return this.$store.dispatch('loadUnreadChats');
            },
            startHistoryPolling() {
                this.pollIntervalId = setInterval(() => {
                    this.loadChatHistory();
                    this.loadUnreadChats();
                }, this.pollMs);
            },
            stopHistoryPolling() {
                if (this.pollIntervalId) {
                    clearInterval(this.pollIntervalId);
                }
            },
            loadChatHistory() {
                if (!this.selectedChat) {
                    return;
                }

                return this.$store.dispatch('loadChatHistory', this.selectedChat);
            },
            getChatTitle(user) {
                if (!user) {
                    return '@';
                }
                return user.first_name
                    ? user.first_name+(user.last_name ? ' '+user.last_name : '')
                    : '@'+(user.username || user.id)
            },
            getMessageTime(message) {
                let time = moment.unix(message.date);
                return time.fromNow();
            },
            getStageName(message) {
                let {funnelId, stageId} = message;
                let stage = this.$store.getters['stage/byFunnelAndId'](funnelId, stageId);
                if (stage) {
                    return stage.title;
                }
            },
            sendReply() {
                let key = this.selectedChat.id+':'+this.selectedChat.botId;
                let text = this.reply[key];
                let funnelId = this.selectedFunnel ? this.selectedFunnel.id : null;
                this.reply[key] = '';

                this.$store.dispatch('reply', {id: this.selectedChat.id, botId: this.selectedChat.botId, funnelId, text})
            },
            markRead(chat) {
                this.selectedChatIndex = null;
                this.$store.dispatch('markRead', {chatId: chat.id, botId: chat.botId});
            }
        },
        computed: {
            unreadChats() {
                return this.$store.state.chat.unread;
            },
            selectedChat() {
                return this.selectedChatIndex !== null ? this.unreadChats[this.selectedChatIndex] : null;
            },
            chatMessages() {
                let history = this.$store.state.chat.chatHistory;
                if (this.selectedFunnel) {
                    history = history.filter(chat => chat.funnelId === this.selectedFunnel.id || !chat.funnelId);
                }

                return history;
            },
            activeFunnels() {
                let botIds = this.unreadChats
                    .map(chat => chat.botId)
                    .filter( (id, index, all) =>  all.indexOf(id) === index );

                let bots = this.$store.state.bot.list.filter(bot => botIds.indexOf(bot.botId) !== -1);
                let funnelsIds = bots
                    .reduce( (funnels, bot) => funnels.concat(bot.funnels || []), [] )
                    .filter( (id, index, all) =>  all.indexOf(id) === index );

                let funnels = this.$store.state.funnel.list.filter(funnel => funnelsIds.indexOf(funnel.id) !== -1);

                return funnels;
            },
            selectedFunnel() {
                if (this.activeFunnels.length === 0) {
                    return null;
                }

                if (this.selectedFunnelIndex === null) {
                    return null;
                }

                return this.activeFunnels[this.selectedFunnelIndex];
            }

        }
    }
</script>

<style scoped>
    .footer {
        position: fixed;
        height: 300px;
        bottom: 0;
        padding: 0!important;
    }

    .content-container {
        padding-bottom: 300px;
    }
</style>