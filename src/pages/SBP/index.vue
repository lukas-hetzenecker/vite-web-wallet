<template>
    <div class="SBP-wrapper __wrapper">
        <sec-title></sec-title>

        <loading v-if="loadingToken" class="loading"></loading>

        <div v-if="!loadingToken" class="section">
            <div class="title">{{ $t('SBP.section1.title') }}</div>
            <register :tokenInfo="tokenInfo" :canUseAddr="canUseAddr" :sendTx="sendTx" class="content"></register>
        </div>

        <div v-if="!loadingToken" class="section">
            <div class="title">{{ $t('SBP.section2.title') }}</div>
            <div class="list-content content">
                <list :showConfirm="showConfirm" :tokenInfo="tokenInfo" :sendTx="sendTx"></list>
            </div>
        </div>

        <div v-if="showConfirmType" class="gray-wrapper">
            <confirm :title="$t(`SBP.confirm.${showConfirmType}.title`)" :singleBtn="true"
                     :closeIcon="true" :close="closeConfirm"
                     :leftBtnTxt="$t(`SBP.confirm.${showConfirmType}.btn`)" :leftBtnClick="validTx"
                     :btnUnuse="!!btnUnuse">
                <div v-if="showConfirmType === 'edit'">
                    <div class="input-err" v-show="addrErr">{{ addrErr }}</div>
                    <vite-input v-model="addr" :valid="testAddr"
                                :placeholder="$t(`SBP.confirm.${showConfirmType}.placeholder`)"></vite-input>
                </div>

                <!-- <div v-if="showConfirmType === 'reward'">
                    <div class="row">
                        <div class="row-t">{{ $t(`SBP.confirm.reward.amount`) }}</div>
                        <div class="row-content unuse">{{ activeItem.showAvailableReward }}</div>
                    </div>

                    <div class="row">
                        <div class="row-t">{{ $t(`SBP.confirm.reward.time`) }}</div>
                        <div class="row-content unuse">{{ activeItem.showAvailableRewardOneTx }}</div>
                    </div>

                    <div class="row">
                        <div class="row-t">
                            {{ $t(`SBP.section2.rewardAddr`)  }}
                            <span v-show="addrErr" class="err">{{ addrErr }}</span>
                        </div>
                        <span class="tips" :class="{ 'active': tips }">{{ tips ? $t('SBP.confirm.reward.hint') : '' }}</span>
                        <div class="row-content">
                            <input v-model="addr" @blur="hideTips" @focus="showTips" autocomplete="off" 
                                   :placeholder="$t(`SBP.confirm.${showConfirmType}.placeholder`)" />
                        </div>
                    </div>
                </div> -->
            </confirm>
        </div>
    </div>
</template>

<script>
import secTitle from 'components/secTitle';
import loading from 'components/loading';
import confirm from 'components/confirm';
import { quotaConfirm } from 'components/quota/index';
import viteInput from 'components/viteInput';
import BigNumber from 'utils/bigNumber';
import { address } from 'utils/tools';
import register from './register';
import list from './list';

export default {
    components: {
        secTitle, register, list, loading, confirm, viteInput
    },
    created() {
        this.tokenInfo = viteWallet.Ledger.getTokenInfo();

        if (!this.tokenInfo) {
            this.loadingToken = true;
            viteWallet.Ledger.fetchTokenInfo().then((tokenInfo) => {
                this.loadingToken = false;
                this.tokenInfo = tokenInfo;
            }).catch((err) => {
                console.warn(err);
            });
        }
    },
    destroyed() {
        this.clearAll();
    },
    data() {
        let activeAccount = this.$wallet.getActiveAccount();

        return {
            activeAccount,
            tokenInfo: {},
            loadingToken: false,
            showConfirmType: '',
            activeItem: {},

            loading: false,
            addr: '',
            addrErr: '',
            tips: false
        };
    },
    computed: {
        btnUnuse() {
            return !this.addr || this.loading || this.addrErr;
        },
        regAddrList() {
            return this.$store.getters.regAddrList;
        }
    },
    methods: {
        canUseAddr(nodeName, addr) {
            let usedAddrList = [];
            for (let name in this.regAddrList) {
                let canUseCancelAddr = (name === nodeName);
                this.regAddrList[name].forEach(item => {
                    if (item.isCancel && canUseCancelAddr) {
                        return;
                    }
                    usedAddrList.push(item.nodeAddr);
                });
            }

            return usedAddrList.indexOf(addr) === -1;
        },
        testAddr() {
            if (this.stopWatch) {
                return;
            }

            if (!this.addr || 
                !address.isValidHexAddr(this.addr)) {
                this.addrErr = this.$t('SBP.section1.addrErr');
                return;
            }

            if (!this.canUseAddr(this.activeItem.name, this.addr)) {
                this.addrErr = this.$t('SBP.section1.addrUsed');
                return;
            }

            this.addrErr = '';
        },
        showTips() {
            this.tips = true;
        },
        hideTips() {
            this.tips = false;
        },

        showConfirm(type, activeItem) {
            this.showConfirmType = type;
            this.stopWatch = false;

            if (!this.tokenInfo || !activeItem) {
                return;
            }

            let decimals = this.tokenInfo.decimals;
            let symbol = this.tokenInfo.tokenSymbol;
            activeItem.showAvailableReward = BigNumber.toBasic(activeItem.availableReward , decimals) + ' ' +  symbol;
            activeItem.showAvailableRewardOneTx = BigNumber.toBasic(activeItem.availableRewardOneTx , decimals) + ' ' +  symbol;
            this.activeItem = activeItem;
        },
        closeConfirm() {
            this.showConfirmType = '';
            this.activeItem = null;
            this.clearAll();
        },
        clearAll() {
            this.stopWatch = true;
            this.addr = '';
            this.addrErr = '';
        },

        validTx() {
            this.testAddr();
            if (this.btnUnuse) {
                return;
            }

            let showConfirmType = this.showConfirmType;
            this.showConfirmType = '';

            this.activeAccount.initPwd({
                cancel: () => {
                    this.closeConfirm();
                },
                submit: () => {
                    this.showConfirm(showConfirmType);
                    if (showConfirmType === 'edit') {
                        this.sendUpdateTx();
                    } else {
                        this.sendRewardTx();
                    }
                }
            });
        },
        sendUpdateTx() {
            this.loading = true;

            let nodeName = this.activeItem.name;
            let producer = this.addr;
            this.sendTx({
                producerAddr: producer
            }, 'updateReg').then(() => {
                this.loading = false;
                this.$toast(this.$t('SBP.section2.updateSuccess'));
                this.closeConfirm();
                this.$store.dispatch('loopRegList', {
                    address: this.activeAccount.getDefaultAddr(),
                    nodeName,
                    operate: 2, 
                    producer
                });
            }).catch((err) => {
                console.warn(err);
                this.loading = false;
                if (err && err.error && err.error.code && err.error.code === -35002) {
                    quotaConfirm({
                        operate: this.$t('SBP.edit')
                    });
                    return;
                }
                this.$toast(this.$t('SBP.section2.updateFail'), err);
            });
        },
        sendRewardTx() {
            this.loading = true;

            this.sendTx({
                rewardAddress: this.addr
            }, 'retrieveReward').then(() => {
                this.loading = false;
                this.$toast(this.$t('SBP.section2.rewardSuccess'));
                this.closeConfirm();
            }).catch((err) => {
                console.warn(err);
                this.loading = false;

                if (err && err.error && err.error.code && err.error.code === -35002) {
                    quotaConfirm({
                        operate: this.$t('SBP.reward')
                    });
                    return;
                }
                this.$toast(this.$t('SBP.section2.rewardFail'), err);
            });
        },

        sendTx({
            producerAddr, rewardAddress, nodeName, amount
        }, type) {
            if (!viteWallet.Net.getNetStatus()) {
                this.$toast(this.$t('nav.noNet'));
                return Promise.reject(false);
            }
          
            let toAmount = BigNumber.toMin(amount || 0, this.tokenInfo.decimals);
            return this.activeAccount[type]({
                tokenId: this.tokenInfo.tokenId,
                nodeName: nodeName || this.activeItem.name,
                amount: toAmount,
                toAddress: producerAddr || rewardAddress
            });
        }
    }
};
</script>

<style lang="scss" scoped>
@import '~assets/scss/table.scss';

.SBP-wrapper {
    position: relative;
    box-sizing: border-box;
    overflow: auto;
    height: 100%;
    .loading {
        width: 60px;
        height: 60px;
        margin-top: -30px;
        margin-left: -30px;
    }
}

.gray-wrapper {
    position: fixed;
    top: 0;
    bottom: 0;
    right: 0;
    left: 0;
    overflow: auto;
    display: flex;
    justify-content: center;
    align-items: center;
    background: rgba(0, 0, 0, 0.6);
    z-index: 100;
    .input-err {
        position: absolute;
        right: 30px;
        top: 2px;
        font-size: 12px;
        color: #FF2929;
        line-height: 26px;
    }
}

.section {
    padding-top: 40px;
    .title {
        border-left: 2px solid rgba(0, 122, 255, 0.7);
        font-family: $font-bold, arial, sans-serif;
        font-size: 18px;
        color: #1d2024;
        line-height: 18px;
        height: 18px;
        margin-bottom: 28px;
        padding-left: 10px;
        margin-bottom: 25px;
    }
    .content {
        background: #FFFFFF;
        border: 1px solid #F6F5F5;
        box-shadow: 0 2px 48px 1px rgba(176,192,237,0.42);
        border-radius: 2px;
    }
    .list-content {
        width: 100%;
        overflow: auto;
    }
}

.row {
    position: relative;
    margin-top: 20px;
    &:first-child {
        margin-top: 0;
    }
    .row-t {
        position: relative;
        font-family: $font-bold, arial, sans-serif;
        font-size: 14px;
        color: #1D2024;
        letter-spacing: 0.35px;
        line-height: 16px;
        padding-bottom: 15px;
    }
    .row-content {
        padding: 10px 15px;
        border: 1px solid #D4DEE7;
        border-radius: 2px;
        font-size: 14px;
        box-sizing: border-box;
        &.unuse {
            background: #F3F6F9;
            font-size: 14px;
            color: #5E6875;
            font-family: $font-normal, arial, sans-serif;
        }
        input {
            width: 100%;
        }
    }
    .err {
        position: absolute;
        left: 90px;
        right: 0;
        font-size: 12px;
        color: #FF2929;
        line-height: 16px;
        text-align: right;
    }
}

@media only screen and (max-width: 550px) {
    .SBP-wrapper {
        padding: 15px;
    }
    .section {
        padding-top: 20px;
    }
}
</style>

<style lang="scss">
@import "~assets/scss/vars.scss";

.tips {
    position: absolute;
    left: 50%;
    bottom: 52px;
    transform: translate(-50%, 0);
    background: #fff;
    box-shadow: 0 5px 20px 0 rgba(0,0,0,0.10);
    border-radius: 8px;
    font-size: 14px;
    color: #3E4A59;
    box-sizing: border-box;
    font-family: $font-normal, arial, sans-serif;
    opacity: 0;
    transition: opacity 0.5s ease-in-out;
    width: 0;
    height: 0; 
    &.active {
        min-width: 300px;
        height: auto;
        opacity: 1;
        padding: 13px 10px;
    }
    &:after {
        content: ' ';
        display: inline-block;
        border: 6px solid transparent;
        border-top: 6px solid #fff;
        position: absolute;
        bottom: -12px;
        left: 50%;
        margin-left: -6px;
    }
}
</style>
