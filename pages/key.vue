<template>
    <b-container class="white-box">
        <div v-if="publicKey && privateKey" class="lead">
            <div class="mb-3" v-if="keyInfo">
                <h3 class="text-center">Thông tin về khóa của bạn</h3>
                <p>Tên: <strong>{{keyInfo.name}}</strong></p>
                <p>Email: <strong>{{keyInfo.email}}</strong></p>
            </div>
            <b-button-toolbar aria-label="Toolbar with button groups and dropdown menu">
                <b-button-group class="mx-1">
                    <b-btn :block="true" :size="'lg'" :variant="'primary'" v-b-modal.modalPrevent>Tạo key mới</b-btn>
                </b-button-group>
                <b-button-group class="mx-1">
                    <b-btn @click="exportKey()">Export</b-btn>
                </b-button-group>
            </b-button-toolbar>
        </div>
        <div v-else>
            <b-btn :block="true" :size="'lg'" :variant="'primary'" v-b-modal.modalPrevent>Tạo Key</b-btn>
            <b-form-file class="mt-3" plain @change="loadTextFromFile"></b-form-file>
        </div>
        <b-modal id="modalPrevent" ref="modal" title="Nhập mật khẩu" @ok="handleOk" @shown="clearName">
            <form @submit.stop.prevent="handleSubmit">
                <b-form-input type="text" placeholder="Nhập mật khẩu cho khóa của bạn"
                              v-model="passphrase"></b-form-input>
            </form>
        </b-modal>
    </b-container>
</template>

<script>
    const openpgp = require('openpgp')

    export default {
        name: 'key',

        data() {
            return {
                publicKey: null,
                privateKey: null,
                file: [],
                keyInfo: null,
                passphrase: null
            }
        },

        methods: {
            async generateKey() {
                let options = {
                    userIds: [{name: this.$auth.user.name, email: this.$auth.user.email}],
                    numBits: 2048,
                    passphrase: this.passphrase
                }
                await openpgp.generateKey(options).then(key => {
                    this.privateKey = key.privateKeyArmored
                    this.publicKey = key.publicKeyArmored
                    localStorage.removeItem('email')
                    localStorage.removeItem('privateKey')
                    localStorage.removeItem('publicKey')
                    localStorage.setItem('email', this.$auth.user.email)
                    localStorage.setItem('privateKey', this.privateKey)
                    localStorage.setItem('publicKey', this.publicKey)
                    this.$axios.post('/api/key', {email: this.$auth.user.email, key: this.publicKey}).then(res => {
                        console.log(res)
                    })
                    this.$toasted.show('Tạo key mới thành công!', {
                        theme: 'bubble',
                        position: 'top-center',
                        duration: 1000
                    })
                })
                this.getKeyInfo()
            },

            exportKey() {
                let FileSaver = require('file-saver')
                let text = this.publicKey + '\n' + this.privateKey
                let blob = new Blob([text], {type: 'text/plain;charset=utf-8'})
                FileSaver.saveAs(blob, 'key.asc')
            },

            loadTextFromFile(ev) {
                const file = ev.target.files[0]
                const reader = new FileReader()
                reader.onload = e => {
                    let result = e.target.result
                    let regex = /-----BEGIN PGP PRIVATE KEY BLOCK-----/i
                    let index = regex.exec(result).index - 1
                    this.publicKey = result.substring(0, index)
                    this.privateKey = result.substring(index + 1, result.length - 1)
                    // console.log(this.publicKey)
                    localStorage.setItem('email', this.$auth.user.email)
                    localStorage.setItem('privateKey', this.privateKey)
                    localStorage.setItem('publicKey', this.publicKey)
                    this.getKeyInfo()
                }
                reader.readAsText(file)
            },

            getKeyInfo() {
                if (this.privateKey) {
                    let keyring = new openpgp.Keyring()
                    let importResult = keyring.privateKeys.importKey(this.privateKey)
                    if (importResult === null) {
                        keyring.store()
                    }
                    let keys = keyring.privateKeys.keys
                    this.keyInfo = keys[0].users[0].userId
                }
            },

            clearName() {
                this.name = ''
            },

            handleOk(evt) {
                // Prevent modal from closing
                evt.preventDefault()
                if (!this.passphrase) {
                    alert('Bạn cần nhập mật khẩu khóa để tiếp tục!')
                } else {
                    this.handleSubmit()
                }
            },

            handleSubmit() {
                this.generateKey()
                this.$refs.modal.hide()
            }
        },

        created() {
            if (process.browser && this.$auth.user) {
                let email = localStorage.getItem('email')
                let privateKey = localStorage.getItem('privateKey')
                let publicKey = localStorage.getItem('publicKey')
                if (this.$auth.user.email === email) {
                    this.publicKey = publicKey
                    this.privateKey = privateKey
                }
            }
        },

        mounted() {
            this.getKeyInfo()
        }
    }
</script>

<style scoped>

</style>
