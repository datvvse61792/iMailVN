<template>
    <div class="cover-container d-flex w-100 h-100 p-3 mx-auto flex-column">
        <header class="masthead">
            <div class="inner">
                <h3 class="masthead-brand">
                    <nuxt-link to="/">Encrypt Mail</nuxt-link>
                </h3>
                <nav class="nav nav-masthead justify-content-center">
                    <nuxt-link to="/" class="nav-link active" v-if="!$auth.user">Trang chủ</nuxt-link>
                    <nuxt-link to="/" class="nav-link" v-else>
                        <i class="mdi mdi-light mdi-email"></i>
                        <span class="has-icon-left">Hòm thư</span>
                    </nuxt-link>
                    <nuxt-link to="/key" class="nav-link" v-if="$auth.user">
                        <i class="mdi mdi-light mdi-key"></i>
                        <span class="has-icon-left">PGP Key</span>
                    </nuxt-link>
                    <nuxt-link to="/logout" class="nav-link" v-if="$auth.user">
                        <i class="mdi mdi-light mdi-bell"></i>
                        <span class="has-icon-left">Thoát</span>
                    </nuxt-link>
                </nav>
            </div>
        </header>
        <nuxt/>
        <footer class="mastfoot mt-auto">
            <div class="inner">
                <p class="mt-5 mb-3 text-muted">2018 &copy; FPT University</p>
            </div>
        </footer>
    </div>
</template>

<style lang="scss">
    .has-icon-left {
        margin-left: 10px;
    }
</style>
