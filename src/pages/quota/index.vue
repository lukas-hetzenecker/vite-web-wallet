<template>
    <div class="quota-wrapper">
        <loading v-if="loadingToken" class="loading"></loading>

        <div v-if="showConfirmType" class="gray-wrapper">
            <confirm v-if="showConfirmType === 'submit' || showConfirmType === 'cancel'" 
                     :title="$t(`quota.confirm.${ showConfirmType }.title`)" :closeIcon="false"
                     :leftBtnTxt="$t(`quota.confirm.${ showConfirmType }.leftBtn`)" :leftBtnClick="closeConfirm"
                     :rightBtnTxt="$t(`quota.confirm.${ showConfirmType }.rightBtn`)" 
                     :rightBtnClick="submit" :rightBtnUnuse="!!cancelUnuse">
                {{ $t(`quota.confirm.${ showConfirmType }.describe`, {amount: activeAmountLimit}) }}
                <div class="cancel-amount" v-show="amountErr">{{ amountErr }}</div>
                <div v-show="showConfirmType === 'cancel'" class="cancel-input">
                    <input type="text" v-model="cancelAmount"
                           :placeholder="$t('quota.cancelAmount')" />
                </div>
            </confirm>

            <confirm  v-if="showConfirmType === 'help'" :title="$t('quota.Q1')" 
                      :closeIcon="false" :singleBtn="true"
                      :leftBtnTxt="$t(`quota.confirm.help.btn`)" :leftBtnClick="closeConfirm">
                <div class="help-t">{{ $t(`quota.confirm.help.t1`) }}</div>
                <div class="help-txt">{{ $t(`quota.confirm.help.txt1`) }}</div>
                <div class="help-t">{{ $t(`quota.confirm.help.t2`) }}</div>
                <div class="help-txt">{{ $t(`quota.confirm.help.txt2`) }}</div>
                <div class="help-t">{{ $t(`quota.confirm.help.t3`) }}</div>
                <div class="help-txt">{{ $t(`quota.confirm.help.txt3`) }}</div>
            </confirm>

            <pow-process ref="powProcess" v-if="showConfirmType === 'pow'"></pow-process>
        </div>

        <div v-show="!loadingToken">
            <quota-head :showHelp="showHelp"></quota-head>
            
            <div class="content">
                <my-quota class="my-quota _content_border"></my-quota>
                <pledge-tx ref="submitPledge" class="pledge-tx _content_border"
                           :tokenInfo="tokenInfo" :sendPledgeTx="sendPledgeTx"
                           :showConfirm="showConfirm" :testAmount="testAmount"></pledge-tx>
            </div>

            <list ref="txList" :sendPledgeTx="sendPledgeTx" 
                  :tokenInfo="tokenInfo"
                  :showConfirm="showConfirm" :testAmount="testAmount"></list>
        </div>
    </div>
</template>

<script>
import Vue from 'vue';
import quotaHead from './quotaHead';
import myQuota from './myQuota';
import pledgeTx from './pledgeTx';
import list from './list';
import confirm from 'components/confirm';
import powProcess from 'components/powProcess';
import loading from 'components/loading';

let amountTimeout = null;

export default {
    components: {
        quotaHead, myQuota, pledgeTx, confirm, list, powProcess, loading
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
        clearTimeout(amountTimeout);
    },
    data() {
        let activeAccount = viteWallet.Wallet.getActiveAccount();

        return {
            activeAccount,
            tokenInfo: {},
            loadingToken: false,
            showConfirmType: '',

            activeAmountLimit: '',
            cancelAmount: '',
            amountErr: '',
            stopWatch: false
        };
    },
    computed: {
        cancelUnuse() {
            return this.showConfirmType === 'cancel' && (!this.cancelAmount || this.amountErr);
        }
    },
    watch: {
        cancelAmount: function() {
            clearTimeout(amountTimeout);
            amountTimeout = null;
            if (this.stopWatch) {
                return;
            }

            amountTimeout = setTimeout(()=> {
                amountTimeout = null;
                this._testAmount();
            }, 500);
        }
    },
    methods: {
        testAmount(amount) {
            return /(^(\d+)$)|(^(\d+[.]\d{1,8})$)/g.test(amount);
        },
        _testAmount() {
            let result = this.testAmount(this.cancelAmount);
            if (!result) {
                this.amountErr = this.$t('transList.valid.amt');
                return false;
            }
            if (viteWallet.BigNumber.isEqual(this.cancelAmount, 0) || 
                viteWallet.BigNumber.compared(this.cancelAmount, this.activeAmountLimit) > 0) {
                this.amountErr = this.$t('quota.maxAmt', {
                    amount: this.activeAmountLimit
                });
                return false;
            }
            this.amountErr = '';
            return true;
        },

        showHelp() {
            this.showConfirmType = 'help';
        },
        showConfirm(type, activeAmountLimit) {
            this.showConfirmType = type || 'submit';
            if (!activeAmountLimit) {
                return;
            }
            this.activeAmountLimit = activeAmountLimit;
        },
        closeConfirm() {
            if (this.showConfirmType === 'cancel') {
                this.stopWatch = true;
                clearTimeout(amountTimeout);
                this.cancelAmount = '';
                this.amountErr = '';
                Vue.nextTick(() => {
                    this.stopWatch = false;
                });
            }
            this.showConfirmType = '';
        },
        submit() {
            if (this.showConfirmType === 'submit') {
                this.closeConfirm();
                let submitPledgeEle = this.$refs.submitPledge;
                submitPledgeEle && submitPledgeEle._sendPledgeTx();
                return;
            }

            this._testAmount();
            if (this.amountErr) {
                return;
            }
            let txListEle = this.$refs.txList;
            if (!txListEle) {
                return;
            }
            txListEle._sendCancelPledgeTx(this.cancelAmount);
            this.closeConfirm();
        },
        stopPow(cb) {
            let powProcessEle = this.$refs.powProcess;
            if (!powProcessEle) {
                return;
            }

            powProcessEle.gotoFinish();
            setTimeout(() => {
                this.closeConfirm();
                cb && cb();
            }, 1000);
        },

        sendPledgeTx({
            toAddr, amount
        }, type, cb) {
            if (!viteWallet.Net.getNetStatus()) {
                this.$toast(this.$t('nav.noNet'));
                cb && cb(false);
                return;
            }

            amount = viteWallet.BigNumber.toMin(amount || 0, this.tokenInfo.decimals);            
            this.activeAccount.sendTx({
                tokenId: this.tokenInfo.tokenId,
                toAddr,
                amount
            }, type).then(() => {
                cb && cb(true);
            }).catch((err) => {
                if (err && err.error && err.error.code && err.error.code === -35002) {
                    this.startPowTx({
                        toAddr, amount
                    }, type, cb);
                    return;
                }
                cb && cb(false);
            });
        },
        startPowTx({
            toAddr, amount
        }, type, cb) {
            this.showConfirm('pow');

            this.activeAccount.getPowTxBlock({
                tokenId: this.tokenInfo.tokenId,
                toAddr,
                amount
            }, type).then((block) => {
                this.stopPow(() => {
                    this.activeAccount.sendRawTx(block).then(() => {
                        cb && cb(true);
                    }).catch(() => {
                        cb && cb(false);
                    });
                });
            }).catch(() => {
                this.closeConfirm();
                cb && cb(false);
            });
        }
    }
};
</script>

<style lang="scss" scoped>
@import "~assets/scss/vars.scss";

.quota-wrapper {
    position: relative;
    padding: 40px;
    box-sizing: border-box;
    overflow: auto;
    height: 100%;
    .loading {
        width: 60px;
        height: 60px;
        margin-top: -30px;
        margin-left: -30px;
    }
    .cancel-amount {
        position: absolute;
        right: 30px;
        font-size: 12px;
        color: #FF2929;
        line-height: 26px;
    }
    .cancel-input {
        background: #FFFFFF;
        border: 1px solid #D4DEE7;
        border-radius: 2px;
        height: 40px;
        margin-top: 27px;
        line-height: 40px;
        input {
            width: 100%;
            text-indent: 15px;
            font-size: 14px;
        }
    }
    .help-t {
        font-family: $font-bold;
        font-size: 16px;
        line-height: 16px;
        margin-bottom: 16px;
        word-break: break-all;
    }
    .help-txt {
        opacity: 0.66;
        font-size: 14px;
        color: #172c39;
        line-height: 22px;
        margin-bottom: 10px;
        word-break: break-all;
    }
}

.content {
    width: 100%;
    display: flex;
    justify-content: space-between;
    flex-wrap: wrap;
    margin-bottom: 40px;
    ._content_border {
        padding: 30px;
        background: #FFFFFF;
        border: 1px solid #F6F5F5;
        box-shadow: 0 2px 48px 1px rgba(176,192,237,0.42);
        border-radius: 2px;
    }
    .my-quota {
        box-sizing: border-box;
        min-width: 170px;
        margin-right: 40px;
    }
    .pledge-tx {
        flex: 1;
        max-width: 100%;
        box-sizing: border-box;
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
}

@media only screen and (max-width: 500px) {
    .content ._content_border {
        padding: 15px;
    }
    .content {
        margin-bottom: 20px;
    }
    .quota-wrapper {
        padding: 15px;
    }
}

@media only screen and (max-width: 950px) {
    .content .my-quota {
        margin-right: 0;
        width: 100%;
        margin-bottom: 20px;
    }
}
</style>