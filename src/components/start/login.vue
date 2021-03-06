<template>
    <div class="login-wrapper">
        <div class="__title">{{$t('nav.head.login')}}</div>

        <div class="bottom __btn __pointer">
            <div v-click-outside="hideAccountList" @click="toggleAccountList">
                <div v-show="!activeAccount" class="__btn_input">{{ $t('create.choose') }}</div>

                <div v-show="activeAccount && !activeAccount.addr" class="__btn __btn_input">
                    <div class="name __ellipsis">{{activeAccount.name}}</div>
                </div>

                <div v-show="activeAccount && activeAccount.addr" class="__btn __btn_input_active">
                    <div class="name __ellipsis">{{activeAccount.name}}</div>
                    <div class="address __ellipsis">{{activeAccount.showAddr}}</div>
                </div>

                <span :class="{ 
                    'slide': true,
                    'down': !isShowAccountList,
                    'up': isShowAccountList
                }"></span>
            </div>
            
            <account-list v-show="isShowAccountList" 
                          :accountList="accountList"
                          :clickAccount="chooseAccount"></account-list>
        </div>

        <div class="bottom __btn __btn_input" 
             :class="{ 'active': !!password || inputItem === 'pass' }">
            <input ref="passInput" autofocus :placeholder="$t('create.input')" 
                   v-model="password" :type="'password'"
                   @focus="inputFocus('pass')" @blur="inputBlur('pass')" />
        </div>

        <div class="bottom __btn __pointer __btn_all_in" @click="login">
            <span v-show="!isLoading">{{ $t('btn.login') }}</span>
            <loading v-show="isLoading" loadingType="dot"></loading>
        </div>

        <div class="btn-list" :class="{ zh: $t('lang') === '中文' }">
            <router-link class="__btn_link" :class="{
                en: $t('lang') === 'English'  
            }" :to="{ name: 'importAccount' }">{{ $t('btn.imported') }}</router-link>
            <span class="line" v-show="$t('lang') === '中文'"></span>
            <router-link class="__btn_link" :class="{
                en: $t('lang') === 'English'  
            }" :to="{ name: 'restore' }">{{$t('mnemonic.restore')}}</router-link>
        </div>
    </div>
</template>

<script>
import Vue from 'vue';
import accountList from './accountList.vue';
import ellipsisAddr from 'utils/ellipsisAddr.js';
import loading from 'components/loading.vue';

export default {
    components: {
        accountList, loading
    },
    mounted() {
        this.$onEnterKey(() => {
            this.login();
        });
        this.activeAccount = this.getLoginAcc();
    },
    destroyed() {
        this.password = '';
        this.isLoading = false;
        this.$offEnterKey();
    },
    data() {
        return {
            activeAccount: {},
            password: '',
            isLoading: false,
            accountList: [],
            isShowAccountList: false,
            inputItem: '',
        };
    },
    methods: {
        getLoginAcc() {
            let account = this.$wallet.getLast();
            if (account) {
                let addr = account.addr || '';
                let showAddr = account.addr ? ellipsisAddr(account.addr) : '';
                return {
                    name: account.name || '',
                    addr,
                    showAddr,
                    id: account.id || ''
                };
            }

            let list = this.$wallet.getList();
            if (!list || !list.length) {
                this.$router.push({
                    name: 'index'
                });
                return;
            }

            account = list[0];
            account.showAddr = account.addr ? ellipsisAddr(account.addr) : '';
            return account;
        },

        inputFocus(text) {
            this.inputItem = text;
        },
        inputBlur(text) {
            text === this.inputItem && (this.inputItem = '');
        },
        focusPass() {
            Vue.nextTick(()=>{
                this.$refs.passInput && this.$refs.passInput.focus();
            });
        },

        chooseAccount(account) {
            this.activeAccount = account;
            this.isShowAccountList = false;
            this.password = '';
        },
        toggleAccountList() {
            this.isShowAccountList = !this.isShowAccountList;
        },
        hideAccountList() {
            this.isShowAccountList = false;
        },

        login() {
            if (!this.activeAccount || this.isLoading) {
                return;
            }

            if (!this.password) {
                this.$toast(this.$t('create.input'), 'error');
                this.focusPass();
                return;
            }

            let loginSuccess = () => {
                if (!this.isLoading) {
                    return;
                }

                this.isLoading = false;
                let activeAccount = this.$wallet.getActiveAccount();
                activeAccount.unlock();
                this.$router.push({ name: 'account' });
            };

            this.isLoading = true;
            this.$wallet.login(this.activeAccount, this.password).then((result) => {
                result && loginSuccess();
                !result && this.$toast(this.$t('hint.pwErr'));
            }).catch((err) => {
                console.warn(err);
                if (!this.isLoading) {
                    return;
                }
                this.isLoading = false;
                this.$toast(this.$t('hint.pwErr'));
            });
        }
    }
};
</script>

<style lang="scss" scoped>
.login-wrapper {
    .__btn {
        position: relative;
        &.__btn_input {
            .name {
                width: 89%;
            }
        }
    }
    .bottom {
        margin-bottom: 20px;
    }
    .slide {
        display: inline-block;
        position: absolute;
        top: 50%;
        right: 20px;
        width: 16px;
        height: 16px;
        margin-top: -6px;
        &.down {
            background: url('../../assets/imgs/down_icon.svg');
            background-size: 16px 16px;
        }
        &.up {
            background: url('../../assets/imgs/up_icon.svg');
            background-size: 16px 16px;
        }
    }

    .btn-list {
        width: 100%;
        text-align: center;
        &.zh {
            height: 20px;
            line-height: 20px;
        }
        .line {
            margin: 0 33px;
            display: inline-block;
            width: 1px;
            height: 100%;
            background: #E5EDF3;
            opacity: 0.3;
            margin-bottom: -4px;
        }
        .__btn_link.en:first-child {
            display: block;
            margin-bottom: 10px;
        }
    }
}
</style>

<style lang="scss">
@import "~assets/scss/vars.scss";
.__btn_input_active {
    border: 1px solid #D4DEE7;
    padding: 8px 40px 8px 20px;
    text-align: left;
    .name {
        font-family: $font-bold, arial, sans-serif;
        font-size: 14px;
        color: #333333;
        line-height: 20px;
    }
    .address {
        font-family: $font-normal-b, arial, sans-serif;
        font-size: 12px;
        line-height: 20px;
        color: #333333;
    }
    background: #fff;
}
</style>
