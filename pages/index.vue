<template>
    <b-container class="mail-box" v-if="$auth.user">
        <b-row>
            <b-col md="3" sm="12">
                <div class="panel">
                    <div class="composer pad-all bord-btm">
                        <b-btn @click="showModal('editor')" class="btn btn-block btn-success" variant="success">Soạn thư
                        </b-btn>
                    </div>
                    <div class="list-group bg-trans pad-btm bord-btm white-panel">
                        <button class=" list-group-item btn btn-link"
                                @click="fetchWithLabel('INBOX', ''), activate(1)"
                                :class="{ active : active_el == 1 }">
                            <i class="mdi mdi-dark mdi-inbox-arrow-down"></i>
                            Hòm thư
                        </button>
                        <button class=" list-group-item btn btn-link"
                                @click="fetchWithLabel('DRAFT', ''), activate(2)"
                                :class="{ active : active_el == 2 }">
                            <i class="mdi mdi-dark mdi-email-open"></i>
                            Nháp
                        </button>
                        <button class=" list-group-item btn btn-link"
                                @click="activate(3), fetchWithLabel('SENT'), resetPageIndex()"
                                :class="{ active : active_el == 3 }">
                            <i class="mdi mdi-dark mdi-inbox-arrow-up"></i>
                            Đã gửi
                        </button>
                        <button class=" list-group-item btn btn-link"
                                @click="fetchWithLabel('SPAM', ''), activate(4), resetPageIndex()"
                                :class="{ active : active_el == 4 }">
                            <i class="mdi mdi-dark mdi-alien"></i>
                            Spam
                        </button>
                        <button class="list-group-item btn btn-link"
                                @click="fetchWithLabel('TRASH', ''), activate(5), resetPageIndex()"
                                :class="{ active : active_el == 5 }">
                            <i class="mdi mdi-dark mdi-delete"></i>
                            Thùng rác
                        </button>
                    </div>
                </div>
            </b-col>
            <b-col md="9" sm="12">
                <div class="white-panel">
                    <div class="panel-header">
                        <b-navbar toggleable="md" variant="faded" type="light">
                            <b-navbar-brand href="#">
                                <h2>{{activeLabel}}
                                </h2>
                            </b-navbar-brand>
                            <b-navbar-nav right>
                                <b-button size="sm" class="my-2 my-sm-0" variant="success" @click="previousPage">
                                    <i class="mdi mdi-light mdi-arrow-left"></i>
                                </b-button>
                                <b-button size="sm" class="my-2 my-sm-0" variant="success" @click="nextPage">
                                    <i class="mdi mdi-light mdi-arrow-right"></i>
                                </b-button>
                            </b-navbar-nav>
                        </b-navbar>
                    </div>
                    <div class="mail-list has-text-left">
                        <b-table :fields="fields" :items="displayEmail">
                            <template slot="title" slot-scope="data">
                                <a href="javascript:void(0);"><h5 @click="readEmail(data.item)">
                                    <span class="icon left">
                                        <i class="mdi mdi mdi-email"></i>
                                    </span>
                                    {{getField(data.item, 'Subject')}}
                                </h5></a>
                            </template>
                            <template slot="sender" slot-scope="data">
                                {{getField(data.item, 'From')}}
                            </template>
                            <template slot="action" slot-scope="data">
                                <b-btn @click="readEmail(data.item)">
                                    <i class="mdi mdi-dark mdi-file-find"></i>
                                </b-btn>
                            </template>
                        </b-table>
                    </div>
                    <div class="paging">

                    </div>
                </div>
            </b-col>
        </b-row>
        <b-modal size="lg" ref="modalShowEditor" @hidden="onHiddenEditor" hide-footer id="modal-center"
                 title="Soạn thư">
            <div class="mb-3">
                <div role="group">
                    <b-input-group prepend="Người nhận">
                        <b-form-input v-model="receiverEmail"></b-form-input>
                    </b-input-group>
                </div>
            </div>
            <div class="mb-3">
                <b-input-group prepend="Tiêu đề">
                    <b-form-input v-model="subject"></b-form-input>
                </b-input-group>
            </div>
            <div class="mb-3">
                <b-form-textarea
                        v-model="message"
                        placeholder="Nội dung"
                        :rows="6"
                        :max-rows="6">
                </b-form-textarea>
            </div>
            <div class="mb-3">
                <p class="btn-group">
                    <button class="btn btn-sm btn-success" @click="encoding">Mã hóa</button>&nbsp;
                    <button class="btn btn-sm btn-success" @click="sendMail(false)">Gửi</button>
                </p>
            </div>
        </b-modal>
        <b-modal size="lg" ref="modalShowMail" @hidden="onHiddenEmail" hide-footer id="show-mail" title="Xem thư">
            <show-mail :mail="activeMail" @delete="mailDelete"></show-mail>
        </b-modal>
    </b-container>

    <b-container class="white-box" v-else>
        <h1 class="cover-heading">Chào mừng bạn đến với iMail.</h1>
        <p class="lead">Tại iMail, bạn sẽ hoàn toàn yên tâm sử dụng email của mình mà không cần phải lo lắng đến việc bị
            đánh cắp nội dung. Chúng tôi sẽ giúp bạn mã hóa thư của mình trước khi gửi và đảm bảo sẽ chỉ có người nhận
            đọc được nội dung.</p>
        <p class="lead">
            <button class="btn btn-lg btn-primary btn-block" @click="google()">Đăng nhập Gmail</button>
        </p>
    </b-container>
</template>
<script>
    import Mail from '../components/mail'

    const url = 'https://www.googleapis.com/gmail/v1/users/me/messages?maxResults=5'
    const detailUrl = 'https://www.googleapis.com/gmail/v1/users/me/messages/'
    const sendUrl = 'https://www.googleapis.com/gmail/v1/users/me/messages/send'
    const openpgp = require('openpgp')

    export default {
        components: {
            'show-mail': Mail
        },
        async asyncData({store, app, query}) {
            let nextPageToken = ''
            let emails = []
            let contacts = []
            try {
                await app.$auth.fetchUser()
            } catch (e) {
                console.log(e)
            }
            if (app.$auth.loggedIn) {
                console.log('Đi sâu vào trong đây')
                const mails = await app.$axios.$get(url)
                nextPageToken = mails.nextPageToken
                for (let i in mails.messages) {
                    let mail = await app.$axios.get(detailUrl + mails.messages[i].id + '?format=full')
                    emails.push(mail.data)
                }
            }
            return {
                emails: emails,
                nextPageToken: nextPageToken,
                contacts,
                pageIndex: 5
            }
        },

        data() {
            return {
                contacts: [],
                currentPage: 1,
                customToolbar: [
                    ['bold', 'italic', 'underline'],
                    [{'list': 'ordered'}, {'list': 'bullet'}],
                    ['image', 'code-block']
                ],
                fields: [
                    // A virtual column that doesn't exist in items
                    // A column that needs custom formatting
                    {key: 'title', label: 'Tiêu đề'},
                    // A regular column
                    {key: 'sender', label: 'Người gửi'},
                    // A virtual column made up from two fields
                    {key: 'action', label: ''}
                ],
                activeMail: null,
                activeLabel: 'Hòm thư',
                message: '',
                subject: '',
                receiverEmail: '',
                currentLabel: 'INBOX',
                active_el: 0
            }
        },

        methods: {
            async google() {
                await this.$auth.loginWith('social').then(response => {
                    console.log(response)
                }).catch(e => {
                    console.log(e)
                })
            },

            async fetchMailList(label, action) {
                if (action === 'new') {
                    this.nextPageToken = ''
                }
                let qLabel = ''
                let qPageToken = ''
                if (label !== '') {
                    qLabel = '&labelIds=' + label
                }

                if (this.nextPageToken !== '') {
                    qPageToken = '&pageToken=' + this.nextPageToken
                }
                console.log('Đã vào tới đây và label là ' + label + ' còn đây là token : ' + this.nextPageToken)
                let query = url + qPageToken + qLabel

                let newEmails = []

                await this.$axios.get(query).then(res => {
                    this.nextPageToken = res.data.nextPageToken
                    newEmails = res.data.messages
                })

                for (let i in newEmails) {
                    let mail = await this.$axios.get(detailUrl + newEmails[i].id + '?format=full')
                    this.emails.push(mail.data)
                }
            },

            async fetchWithLabel(label) {
                if (label === '') {
                    await this.fetchMailList(this.currentLabel, 'new')
                } else {
                    this.currentLabel = label
                    await this.fetchMailList(this.currentLabel, '')
                }
            },

            async next() {
                let pageToken = this.pageTokens[this.pageTokens.length - 1]
                await this.fetchWithLabel(this.currentLabel, pageToken)
            },

            async previous() {
                this.pageTokens.splice(this.pageTokens.length - 1, 1)
                let pageToken = ''
                if (this.pageTokens.length > 2) {
                    pageToken = this.pageTokens[this.pageTokens.length - 2]
                }
                await this.fetchWithLabel(this.currentLabel, pageToken)
                if (this.pageTokens.length > 3) {
                    this.pageTokens.splice(this.pageTokens.length - 2, 2)
                }
            },

            async nextPage() {
                if (this.pageIndex === this.emails.length) {
                    await this.fetchWithLabel(this.currentLabel, this.pageIndex)
                }
                this.pageIndex += 5
            },

            async previousPage() {
                if (this.pageIndex === 5 || this.pageIndex < 5) {
                    this.pageIndex = 5
                } else {
                    this.pageIndex -= 5
                }
            },

            mailDelete() {
                for (let index in this.emails) {
                    if (this.emails[index].id === this.activeMail.id) {
                        this.emails.splice(index, 1)
                    }
                }
                this.hideModal('mail')
            },

            readEmail(email) {
                this.activeMail = email
                this.showModal('mail')
            },

            showModal(modal) {
                switch (modal) {
                    case 'editor':
                        this.$refs.modalShowEditor.show()
                        break
                    case 'mail':
                        this.$refs.modalShowMail.show()
                        break
                    default:
                        break
                }
            },

            hideModal(modal) {
                switch (modal) {
                    case 'editor':
                        this.$refs.modalShowEditor.hide()
                        break
                    case 'mail':
                        this.$refs.modalShowMail.hide()
                        break
                }
            },

            getField(data, name) {
                for (let index in data.payload.headers) {
                    let header = data.payload.headers[index]
                    if (header.name === name) {
                        return header.value
                    }
                }
            },

            onHiddenEmail(evt) {
                this.activeMail = null
            },

            sendMail(isEncode) {
                let formData = new FormData()
                formData.append('subject', this.subject)
                formData.append('receiver', this.receiverEmail)
                formData.append('message', this.message)
                if (isEncode) {
                    formData.append('encode', 'yes')
                }

                let raw = this.makeBody(this.receiverEmail, this.$auth.user.email, this.subject, this.message)

                this.$axios.post(sendUrl, {raw: raw}).then(res => {
                    this.message = ''
                    this.$toasted.show('Đã gửi thành công!', {
                        theme: 'bubble',
                        position: 'top-center',
                        duration: 1000
                    })
                }).catch(err => {
                    console.log(err)
                    this.$toasted.show('Gửi lên thất bại!', {
                        theme: 'primary',
                        position: 'top-center',
                        duration: 1000
                    })
                })
            },

            activate: function (el) {
                this.active_el = el
            },

            resetPageIndex(label) {
                this.pageIndex = 5
                this.emails = []
                this.fetchMailList(label, 'new')
                this.nextPageToken = ''
            },

            makeBody(to, from, subject, message) {
                let Buffer = require('safe-buffer').Buffer
                let emailLines = []
                emailLines.push('From: ' + from)
                emailLines.push('To: ' + to)
                emailLines.push('Content-type: text/plain;charset=UTF-8')
                emailLines.push('Content-Transfer-Encoding: 8bit')
                emailLines.push('MIME-Version: 1.0')
                subject = Buffer.from(subject).toString('base64')
                subject = '=?utf-8?B?' + subject + '?='
                emailLines.push('Subject: ' + subject)
                emailLines.push('')
                emailLines.push(message)
                let email = emailLines.join('\r\n').trim()
                let base64EncodedEmail = new Buffer(email, 'UTF-8').toString('base64')
                return base64EncodedEmail.replace(/\+/g, '-').replace(/\//g, '_')
            },
            onHiddenEditor(event) {
                this.subject = ''
                this.receiverEmail = ''
                this.message = ''
            },
            findPublicKey(email) {
                for (let i in this.contacts) {
                    if (this.contacts[i].email === email) {
                        return this.contacts[i].key
                    }
                }
                return null
            },
            encoding() {
                let publicKey = this.findPublicKey(this.receiverEmail)
                if (publicKey) {
                    console.log(publicKey)
                    let options = {
                        data: this.message,
                        publicKeys: openpgp.key.readArmored(publicKey).keys
                    }
                    openpgp.encrypt(options).then(ciphertext => {
                        this.message = ciphertext.data
                        this.$toasted.show('Mã hóa thành công!', {
                            theme: 'bubble',
                            position: 'top-center',
                            duration: 1000
                        })
                    })
                } else {
                    this.$toasted.show('Không thấy email', {
                        theme: 'primary',
                        position: 'top-center',
                        duration: 1000
                    })
                }
            }
        },

        computed: {
            displayEmail() {
                return this.emails.slice(this.pageIndex - 5, this.pageIndex)
            }
        },
        async created() {
            await this.$axios.get('/api/key').then(res => {
                this.contacts = res.data
            })
        }
    }
</script>

<style lang="scss">
    .mail-box {
        .composer {
            margin-bottom: 10px;
        }
        padding-left: 0;
        padding-right: 0;
        color: black;
        .btn-success {
            color: white;
        }
        .white-panel {
            background: white;
            color: black;
            a {
                color: #333333;
            }
        }
        .paging {
            padding: 10px;
        }
        .list-group {
            .btn {
                text-align: left;
            }
        }
    }
</style>
