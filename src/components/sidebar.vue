<template>
    <div class="sidebar-wrapper">
        <div @mouseenter="overLogo"  @mouseleave="leaveLogo" class="logo __pointer">
            <img :src="viteLogo" />
            <test-notice class="notice" :class="{'hide': !isShowNotice}"></test-notice>
        </div>

        <router-link v-for="(name, index) in pageList" :key="index"
                     class="__pointer icon" :class="{
                         'active': active === name
        }" :to="{ 'name': name }">
            <img v-show="active !== name" :src="icon[name]" />
            <img v-show="active === name" :src="icon[`${name}Active`]"  />
        </router-link>

        <div class="_bottom">
            <router-link class="icon setting __pointer" :class="{
                'active': active === 'setting'
            }" :to="{ name: 'setting' }">
                <img v-show="active !== 'setting'" :src="setting" />
                <img v-show="active === 'setting'" :src="settingActive"  />
            </router-link>

            <div class="icon logout __pointer" @click="logout" 
                 @mouseenter="enterLogout" @mouseleave="leaveLogout">
                <img v-show="!logoutHover" :src="logoutDefault" />
                <img v-show="logoutHover" :src="logoutActive" />
            </div>
        </div>
    </div>
</template>

<script>
import testNotice from 'components/testNotice';

import viteLogo from 'assets/imgs/sidebar_logo.svg';
import account from 'assets/imgs/index_icon_default.svg';
import accountActive from 'assets/imgs/index_icon_pressed.svg';
import transList from 'assets/imgs/transfer_default.svg';
import transListActive from 'assets/imgs/transfer_pressed.svg';
import setting from 'assets/imgs/settings_default.svg';
import settingActive from 'assets/imgs/settings_pressed.svg';
import logoutDefault from 'assets/imgs/logout_default.svg';
import logoutActive from 'assets/imgs/logout_pressed.svg';
import quota from 'assets/imgs/quota_default.svg';
import quotaActive from 'assets/imgs/quota_pressed.svg';
import SBP from 'assets/imgs/SBP_default.svg';
import SBPActive from 'assets/imgs/SBP_active.svg';
import vote from 'assets/imgs/vote_default.svg';
import voteActive from 'assets/imgs/vote_active.svg';
import exchangeVite from 'assets/imgs/exchangeVite_default.svg';
import exchangeViteActive from 'assets/imgs/exchangeVite_pressed.svg';

export default {
    components: {
        testNotice
    },
    props: {
        active: {
            type: String,
            default: ''
        }
    },
    data() {
        let activeAccount = this.$wallet.getActiveAccount();
        let pageList = ['account', 'quota', 'SBP', 'vote', 'transList'];
        if (activeAccount.type === 'wallet') {
            pageList.push('exchangeVite');
        }

        return {
            isShowNotice: false,
            logoutHover: false,
            viteLogo,
            pageList,
            icon: {
                account,
                accountActive,
                transList,
                transListActive,
                quota,
                quotaActive,
                SBP,
                SBPActive,
                vote,
                voteActive,
                exchangeVite,
                exchangeViteActive
            },
            setting,
            settingActive,
            logoutDefault,
            logoutActive
        };
    },
    methods: {
        overLogo() {
            this.isShowNotice = true;
        },
        leaveLogo() {
            this.isShowNotice = false;
        },

        enterLogout() {
            this.logoutHover = true;
        },
        leaveLogout() {
            this.logoutHover = false;
        },
        logout() {
            this.$router.push({
                name: 'index'
            });
        }
    }
};
</script>

<style lang="scss" scoped>
.sidebar-wrapper {
    position: absolute;
    top: 0;
    bottom: 0;
    width: 100%;
    background: #fff;
    box-shadow: 0 2px 40px 1px rgba(221,229,252,0.50);
    .logo {
        display: inline-block;
        margin-top: 24px;
        width: 100%;
        height: 50px;
        img {
            width: 100%;
            height: 100%;
        }
        .notice {
            transition: opacity 0.5s ease-in-out;
            opacity: 1;
            &.hide {
                width: 0;
                height: 0;
                opacity: 0;
                overflow: hidden;
            }
        }
    }
    .icon {
        position: relative;
        display: flex;
        justify-content: center;
        align-items: center;
        width: 100%;
        height: 40px;
        margin-top: 30px;
        img {
            width: 24px;
            height: 24px;
        }
        &.active:before {
            content: '';
            position: absolute;
            top: 0;
            left: 0;
            bottom: 0;
            right: 4px;
            display: inline-block;
            width: 4px;
            height: 40px;
            background-image: linear-gradient(-90deg, #1B3BD8 100%, #176CE0 100%, #0B92E7 100%, #0BB6EB 100%, #00E0F2 100%);
        }
    }
    ._bottom {
        position: absolute;
        bottom: 50px;
        width: 100%;
        .icon {
            margin-top: 0;
        }
        .setting {
            margin-bottom: 30px;
        }
        .logout {
            height: 30px;
        }
    }
}
</style>
