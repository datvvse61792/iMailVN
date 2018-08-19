<template>
    <div v-if="mail !== null" class="mail-detail">
        <div class="email mb-3">
            <b-media>
                <h5 class="mt-0">{{getField(mail, 'Subject')}}</h5>
                <p>{{getField(mail, 'From')}}</p>
            </b-media>
            <div class="mb-3">
                <div v-if="decodedText" class="alert alert-info">{{decodedText}}</div>
                <div v-else class="content" v-html="getBody(mail)"></div>
            </div>
            <div class="footer">
                <div class="btn-group" role="group" aria-label="First group">
                    <b-btn @click="changeToDecrypt()">
                        <i class="mdi mdi-dark mdi-barcode-scan">&nbsp;Giải mã</i>
                    </b-btn>
                    <b-btn @click="changeToReply()">
                        <i class="mdi mdi-dark mdi-reply">&nbsp;Trả lời</i>
                    </b-btn>
                    <b-btn @click="deleteEmail">
                        <i class="mdi mdi-dark mdi-delete">&nbsp;Xóa</i>
                    </b-btn>
                </div>
            </div>
        </div>
        <div v-if="showReply" class="reply mb-3">
            <h4>Trả lời</h4>
            <div class="mb-3">
                <b-form-textarea
                        v-model="message"
                        placeholder="Nội dung"
                        :rows="6"
                        :max-rows="6">
                </b-form-textarea>
            </div>
            <b-button size="sm" variant="success" @click="encoding">
                Mã Hóa
            </b-button>
            <b-button size="sm" variant="success" @click="reply">
                Gửi thường
            </b-button>
        </div>
        <div v-if="enterPass" class="decode mb-3">
            <h4>Giải mã nội dung thư</h4>
            <div class="mb-3">
                <b-input
                        v-model="keyPassword"
                        placeholder="nhập mật khẩu"
                        type="password">
                </b-input>
            </div>
            <b-button size="sm" variant="success" @click="decodeEmail">
                Giải mã
            </b-button>
        </div>
    </div>
</template>

<script>
    const deleteUrl = 'https://www.googleapis.com/gmail/v1/users/me/messages/'
    const sendUrl = 'https://www.googleapis.com/gmail/v1/users/me/messages/send'
    const openpgp = require('openpgp')

    export default {
        name: 'mail',

        props: {
            mail: {
                type: Object,
                default: null
            }
        },

        watch: {
            mail() {
                this.decodedText = null
            }
        },

        data() {
            return {
                contacts: [],
                customToolbar: [
                    ['bold', 'italic', 'underline'],
                    [{'list': 'ordered'}, {'list': 'bullet'}],
                    ['image', 'code-block']
                ],
                showReply: false,
                enterPass: false,
                message: '',
                keyPassword: '',
                decodedText: null
            }
        },

        methods: {
            reply() {
                let Buffer = require('safe-buffer').Buffer
                let emailLines = []
                let subject = this.getField(this.mail, 'Subject')
                emailLines.push('From: ' + this.$auth.user.email)
                emailLines.push('To: ' + this.getEmailReceiver)
                emailLines.push('Content-type: text/plain;charset=UTF-8')
                emailLines.push('Content-Transfer-Encoding: 8bit')
                emailLines.push('MIME-Version: 1.0')
                console.log('Subject ban đầu: ' + subject)
                subject = Buffer.from(subject).toString('base64')
                console.log('Subject được mã hóa thành: ' + subject)
                subject = '=?utf-8?B?' + subject + '?='
                emailLines.push('Subject: ' + subject)
                emailLines.push('')
                emailLines.push(this.message)
                let email = emailLines.join('\r\n').trim()
                let base64EncodedEmail = new Buffer(email, 'UTF-8').toString('base64')
                let raw = base64EncodedEmail.replace(/\+/g, '-').replace(/\//g, '_')
                let threadId = this.mail.id
                if (this.mail.threadId) {
                    threadId = this.mail.threadId
                }

                this.$axios.post(sendUrl, {raw: raw, threadId: threadId}).then(res => {
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
            async decodeEmail() {
                let publicKey = this.findPublicKey(this.getEmailReceiver)
                let privKeyObj = openpgp.key.readArmored(localStorage.getItem('privateKey')).keys[0]
                if (privKeyObj == null) {
                    this.$toasted.show('Chưa có key. Vui lòng Tạo key hoặc nhập key từ máy tính của bạn', {
                        theme: 'primary',
                        position: 'top-center',
                        duration: 1500
                    })
                } else {
                    let body = this.getBody(this.mail)
                    console.log(body)
                    await privKeyObj.decrypt(this.keyPassword)
                    const options = {
                        message: openpgp.message.readArmored(body),
                        publicKeys: openpgp.key.readArmored(publicKey).keys,
                        privateKeys: [privKeyObj]
                    }
                    openpgp.decrypt(options).then(plaintext => {
                        this.decodedText = plaintext.data
                        this.$toasted.show('Giải mã thành công', {
                            theme: 'bubble',
                            position: 'top-center',
                            duration: 1500
                        })
                    }).catch(err => {
                        console.log(err)
                        this.$toasted.show('Khóa của bạn không hợp lệ', {
                            theme: 'primary',
                            position: 'top-center',
                            duration: 1500
                        })
                    })
                }
            },
            encoding() {
                let publicKey = this.findPublicKey(this.getEmailReceiver)
                if (publicKey) {
                    console.log(publicKey)
                    let options = {
                        data: this.message,
                        publicKeys: openpgp.key.readArmored(publicKey).keys
                    }
                    openpgp.encrypt(options).then(ciphertext => {
                        this.message = ciphertext.data
                    })
                }
            },
            findPublicKey(email) {
                for (let i in this.contacts) {
                    console.log('Day la email : ' + this.contacts[i].email)
                    if (this.contacts[i].email === email) {
                        return this.contacts[i].key
                    }
                }
                console.log('Tao khong tim thay ' + email)
                return null
            },
            deleteEmail() {
                this.$axios.delete(deleteUrl + this.mail.id).then(res => {
                    console.log(res)
                    this.$emit('delete')
                    this.$toasted.show('Xóa thành công', {
                        theme: 'bubble',
                        position: 'top-center',
                        duration: 1500
                    })
                }).catch(err => {
                    console.log(err)
                    this.$toasted.show('Xóa lỗi, làm ơn thử lại!', {
                        theme: 'primary',
                        position: 'top-center',
                        duration: 1500
                    })
                })
            },

            getField(data, name) {
                for (let index in data.payload.headers) {
                    let header = data.payload.headers[index]
                    if (header.name === name) {
                        return header.value
                    }
                }
            },

            getHTMLPart(arr) {
                if (typeof arr === 'object') {
                    for (let x = 0; x <= arr.length; x++) {
                        if (typeof arr[x].parts === 'undefined') {
                            if (arr[x].mimeType === 'text/html') {
                                return arr[x].body.data
                            }
                        } else {
                            return this.getHTMLPart(arr[x].parts)
                        }
                    }
                }
                return ''
            },

            getBody(message) {
                let encodedBody = ''
                if (typeof message.payload.parts === 'undefined') {
                    encodedBody = message.payload.body.data
                } else {
                    encodedBody = this.getHTMLPart(message.payload.parts)
                }
                encodedBody = encodedBody.replace(/-/g, '+').replace(/_/g, '/').replace(/\s/g, '')
                return decodeURIComponent(escape(window.atob(encodedBody)))
            },
            changeToDecrypt() {
                this.enterPass = !this.enterPass
                this.showReply = false
            },

            changeToReply() {
                this.showReply = !this.showReply
                this.enterPass = false
            }
        },

        computed: {
            getEmailReceiver() {
                let re = /<(.*?)>/i
                let from = this.getField(this.mail, 'From')
                let to = re.exec(from)
                if (to) {
                    return to[1]
                } else {
                    return this.getField(this.mail, 'From')
                }
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
    .mail-detail {
        .content {
            overflow: auto;
        }
    }
</style>
