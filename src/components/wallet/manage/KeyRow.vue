<template>
    <div class="addressItem" :selected="is_default">
        <ExportKeys
            v-if="walletType === 'mnemonic'"
            :wallets="[wallet]"
            ref="export_wallet"
        ></ExportKeys>
        <mnemonic-phrase
            v-if="walletType === 'mnemonic'"
            :phrase="mnemonicPhrase"
            ref="modal"
        ></mnemonic-phrase>
        <HdDerivationListModal
            :wallet="wallet"
            ref="modal_hd"
            v-if="isHDWallet"
        ></HdDerivationListModal>
        <PrivateKey
            v-if="walletType === 'singleton'"
            :privateKey="privateKey"
            ref="modal_priv_key"
        ></PrivateKey>
        <div class="rows">
            <div class="header">
                <template v-if="is_default">
                    <img src="@/assets/key_active.svg" class="key_logo" />
                </template>
                <template v-else>
                    <img
                        v-if="$root.theme === 'day'"
                        src="@/assets/key_inactive.svg"
                        class="key_logo"
                    />
                    <img v-else src="@/assets/key_inactive_night.png" class="key_logo" />
                </template>
                <div class="header_cols">
                    <div class="detail">
                        <p class="addressVal">
                            <b>{{ walletTitle }}</b>
                        </p>
                        <Tooltip :text="$t('keys.tooltip')" v-if="isVolatile">
                            <fa icon="exclamation-triangle" class="volatile_alert"></fa>
                        </Tooltip>
                    </div>
                    <div class="buts">
                        <button class="selBut" @click="select" v-if="!is_default">
                            <span>{{ $t('keys.activate_key') }}</span>
                        </button>
                        <Tooltip
                            :text="$t('keys.remove_key')"
                            class="row_but circle"
                            @click.native="remove"
                            v-if="!is_default"
                        >
                            <img src="@/assets/trash_can_dark.svg" style="height: 16px" />
                        </Tooltip>
                        <Tooltip
                            v-if="walletType !== 'singleton'"
                            :text="$t('keys.hd_addresses')"
                            class="row_but circle"
                            @click.native="showPastAddresses"
                        >
                            <fa icon="list-ol"></fa>
                        </Tooltip>
                        <Tooltip
                            v-if="walletType === 'mnemonic'"
                            :text="$t('keys.export_key')"
                            class="row_but circle"
                            @click.native="showExportModal"
                        >
                            <fa icon="upload"></fa>
                        </Tooltip>
                        <button v-if="walletType == 'mnemonic'" @click="showModal">
                            {{ $t('keys.view_key') }}
                        </button>
                        <button v-if="walletType == 'singleton'" @click="showPrivateKeyModal">
                            {{ $t('keys.view_priv_key') }}
                        </button>
                    </div>
                </div>
            </div>

            <div class="header">
                <div></div>
                <div>
                    <p v-if="Object.keys(balances).length === 0" class="balance_empty">
                        {{ $t('keys.empty') }}
                    </p>
                    <div class="addressBalance bal_cols" v-else>
                        <p>This key has:</p>
                        <div class="bal_rows">
                            <p v-for="bal in balances" :key="bal.id">
                                {{ bal.toString() }}
                                <b>{{ bal.symbol }}</b>
                            </p>
                        </div>
                    </div>
                    <div class="addressBalance bal_cols">
                        <p>The extended public key is:</p>
                        <div class="public_key_rows">
                            <p>{{ publicExtendedKey }}</p>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </div>
</template>
<script lang="ts">
import 'reflect-metadata'
import { Vue, Component, Prop } from 'vue-property-decorator'

import { bintools, keyChain } from '@/AVA'
import AvaAsset from '@/js/AvaAsset'
import { AssetsDict } from '@/store/modules/assets/types'
import { AmountOutput, KeyPair as AVMKeyPair } from 'avalanche/dist/apis/avm'
import MnemonicPhrase from '@/components/modals/MnemonicPhrase.vue'
import HdDerivationListModal from '@/components/modals/HdDerivationList/HdDerivationListModal.vue'
import AvaHdWallet from '@/js/wallets/AvaHdWallet'
import Tooltip from '@/components/misc/Tooltip.vue'

import ExportKeys from '@/components/modals/ExportKeys.vue'
import PrivateKey from '@/components/modals/PrivateKey.vue'
import { WalletNameType, WalletType } from '@/store/types'
import { SingletonWallet } from '../../../js/wallets/SingletonWallet'

interface IKeyBalanceDict {
    [key: string]: AvaAsset
}

@Component({
    components: {
        MnemonicPhrase,
        HdDerivationListModal,
        Tooltip,
        ExportKeys,
        PrivateKey,
    },
})
export default class KeyRow extends Vue {
    @Prop() wallet!: WalletType
    @Prop({ default: false }) is_default?: boolean

    get isVolatile() {
        return this.$store.state.volatileWallets.includes(this.wallet)
    }

    get walletTitle() {
        return this.wallet.getBaseAddress()
    }

    get assetsDict(): AssetsDict {
        return this.$store.state.Assets.assetsDict
    }

    get balances(): IKeyBalanceDict {
        if (!this.wallet.getUTXOSet()) return {}

        let res: IKeyBalanceDict = {}

        let addrUtxos = this.wallet.getUTXOSet().getAllUTXOs()
        for (var n = 0; n < addrUtxos.length; n++) {
            let utxo = addrUtxos[n]

            // ignore NFTS and mint outputs
            //TODO: support nfts
            let outId = utxo.getOutput().getOutputID()
            if (outId === 11 || outId === 6 || outId === 10) continue

            let utxoOut = utxo.getOutput() as AmountOutput

            let amount = utxoOut.getAmount()
            let assetIdBuff = utxo.getAssetID()
            let assetId = bintools.cb58Encode(assetIdBuff)

            let assetObj: AvaAsset | undefined = this.assetsDict[assetId]

            if (!assetObj) {
                let name = '?'
                let symbol = '?'
                let denomination = 0

                let newAsset = new AvaAsset(assetId, name, symbol, denomination)
                newAsset.addBalance(amount)

                res[assetId] = newAsset
                continue
            }

            let asset = res[assetId]
            if (!asset) {
                let name = assetObj.name
                let symbol = assetObj.symbol
                let denomination = assetObj.denomination

                let newAsset = new AvaAsset(assetId, name, symbol, denomination)
                newAsset.addBalance(amount)

                res[assetId] = newAsset
            } else {
                asset.addBalance(amount)
            }
        }

        return res
    }

    get walletType(): WalletNameType {
        return this.wallet.type
    }

    get isHDWallet() {
        return ['mnemonic', 'ledger'].includes(this.walletType)
    }
    get mnemonicPhrase(): string {
        if (this.walletType !== 'mnemonic') return '?'
        let wallet = this.wallet as AvaHdWallet
        return wallet.getMnemonic()
    }

    get privateKey(): string {
        if (this.walletType !== 'singleton') return '?'
        let wallet = this.wallet as SingletonWallet
        return wallet.key
    }

    get publicExtendedKey(): string {
        if (this.walletType === 'singleton') return 'No data'
        let wallet = this.wallet as AvaHdWallet
        return wallet.getPublicExtendedKey()
    }

    remove() {
        this.$emit('remove', this.wallet)
    }
    select() {
        this.$emit('select', this.wallet)
    }

    showModal() {
        let modal = this.$refs.modal as MnemonicPhrase
        //@ts-ignore
        modal.open()
    }

    showPastAddresses() {
        let modal = this.$refs.modal_hd as MnemonicPhrase
        //@ts-ignore
        modal.open()
    }

    showExportModal() {
        //@ts-ignore
        this.$refs.export_wallet.open()
    }

    showPrivateKeyModal() {
        //@ts-ignore
        this.$refs.modal_priv_key.open()
    }
}
</script>
<style scoped lang="scss">
@use '../../../main';

.addressItem {
    font-size: 12px;
    /*display: grid;*/
    /*grid-template-columns: 1fr max-content;*/
    /*grid-gap: 15px;*/
    overflow: auto;

    > * {
        align-self: center;
        overflow: auto;
    }
}

.key_logo {
    width: 32px;
}

.hdlist {
    grid-column: 1/3;
}

.buts {
    display: flex;
    align-items: center;
    flex-direction: row;
    flex-wrap: wrap;

    > * {
        margin: 0px 8px !important;
    }

    button {
        font-size: 16px;
    }

    $but_w: 32px;
    .circle {
        width: $but_w;
        height: $but_w;
        border-radius: $but_w;
        background-color: rgba(0, 0, 0, 0.1);
        display: flex;
        justify-content: center;
        align-self: center;

        &:hover {
            background-color: var(--bg);
        }
    }
}

.row_but {
    margin: 0 12px;
}

.rows {
    overflow: auto;
}
.addressItem .selBut {
    flex-shrink: 0;
    flex-grow: 1;
    /*background-color: #C0C0CD;*/
    color: #867e89;
    padding: 4px 8px;

    span {
        font-size: 12px;
        line-height: normal;
    }
}

.addressItem {
    .selBut {
    }
}

.header {
    display: grid;
    grid-template-columns: 32px 1fr;
    grid-gap: 14px;
    /*align-items: center;*/
}

.header_cols {
    display: flex;
    align-items: center;
    justify-content: space-between;
}

.detail {
    overflow: auto;
    display: flex;
    align-items: center;

    /*grid-template-columns: max-content max-content;*/
    /*column-gap: 15px;*/
}

.label {
    font-weight: bold;
}
.addressVal {
    overflow: auto;
    text-overflow: ellipsis;
    white-space: nowrap;
    font-family: monospace;
    font-size: 15px;

    span {
        font-weight: normal;
        margin-right: 8px;
    }
}

.del {
    align-self: start;
    opacity: 0.4;

    &:hover {
        opacity: 1;
    }
}

.addressBalance {
    display: flex;
    white-space: nowrap;
    color: var(--primary-color);
    .bal_rows p {
        font-weight: bold;
        padding: 0px 8px;
        margin-bottom: 4px;
    }
    .public_key_rows p {
        font-size: 8px;
        padding: 4px 0px 0px 8px;
    }
    p {
        border-radius: 3px;
    }
}

.bal_cols {
    display: flex;
}

.bal_rows {
    display: flex;
    flex-direction: column;
}

.balance_empty {
    color: var(--primary-color);
}

.volatile_alert {
    color: var(--warning);
    font-size: 15px;
    margin-left: 6px;
}

@include main.mobile-device {
    .header_cols {
        display: block;
    }

    .detail {
        text-align: right;
    }

    .bal_cols {
        border-top: 1px solid #ddd;
        padding-top: 12px;
        margin-top: 12px;
    }
}
</style>
