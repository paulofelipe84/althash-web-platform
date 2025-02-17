<template>
  <v-card>
    <v-card-title>
      <div class="headline font-weight-medium blue--text">
        <v-icon large>play_circle_outline</v-icon>
          {{ $t('send_to_contract.title') }}
      </div>
    </v-card-title>
    <v-card-text>
      <v-form>
        <v-text-field
          label="Contract Address"
          v-model.trim="contractAddress"
          required
          outline
          background-color="blue lighten-1"
        ></v-text-field>
        <v-textarea
          label="ABI"
          v-model.trim="abi"
          required
          outline
          background-color="blue lighten-1"
          @input="decodeAbi"
        ></v-textarea>
        <v-select
          v-if="parsedAbi"
          :items="parsedAbi"
          label="Method"
          v-model="method"
          outline
          background-color="blue lighten-1"
          single-line
          bottom
        ></v-select>
        <template v-if="params">
          <v-text-field
            v-for="(param, index) in params"
            :label="param.name"
            :key="index"
            v-model="inputParams[index]"
            outline
            background-color="blue lighten-1"
          ></v-text-field>
        </template>
        <v-text-field
          label="Gas Price (1e-8 HTML/gas)"
          v-model.trim="gasPrice"
          outline
          background-color="indigo darken-4"
          required
        ></v-text-field>
        <v-text-field
          label="Gas Limit"
          v-model.trim="gasLimit"
          outline
          background-color="indigo darken-4"
          required
        ></v-text-field>
        <v-text-field
          label="Fee"
          v-model.trim="fee"
          outline
          background-color="indigo darken-4"
          required
          ></v-text-field>
         <v-text-field
          label="Amount"
          v-model.trim="fee"
          outline
          background-color="indigo darken-4"
          required
          ></v-text-field>
      </v-form>
    </v-card-text>
    <v-card-actions>
      <v-spacer></v-spacer>
      <v-btn class="success" dark @click="send" :disabled="notValid">{{ $t('common.confirm') }}</v-btn>
      <v-spacer></v-spacer>
    </v-card-actions>
    <v-dialog v-model="confirmSendDialog" persistent max-width="50%">
      <v-card>
        <v-card-title>
          <span class="headline">
            {{ $t('send_to_contract.confirm') }}
          </span>
        </v-card-title>
        <v-card-text>
          <v-container grid-list-md>
            <v-layout wrap>
              <v-flex xs12>
                <v-textarea label="Raw Tx" v-model="rawTx" disabled></v-text-field>
              </v-flex>
            </v-layout>
          </v-container>
        </v-card-text>
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn class="blue--text darken-1" flat @click="confirmSend" v-show="canSend && !sending">{{ $t('common.confirm') }}</v-btn>
          <v-btn class="red--text darken-1" flat @click.native="confirmSendDialog = false" :v-show="!sending">{{ $t('common.cancel') }}</v-btn>
          <v-progress-circular indeterminate :size="20" v-show="sending" class="primary--text"></v-progress-circular>
        </v-card-actions>
      </v-card>
    </v-dialog>
  </v-card>
</template>

<script>
import webWallet from 'libs/web-wallet'
import abi from 'ethjs-abi'
import server from 'libs/server'

export default {
  data () {
    return {
      contractAddress: '',
      abi: '',
      parsedAbi: null,
      method: null,
      inputParams: [],
      amountContract: '0.001',
      gasPrice: '40',
      gasLimit: '2500000',
      fee: '0.01',
      confirmSendDialog: false,
      rawTx: 'loading...',
      canSend: false,
      sending: false
    }
  },
  computed: {
    params: function() {
      if (this.method === null) {
        return null
      }
      const inputs = this.parsedAbi[this.method].info.inputs
      if (inputs.length > 0) {
        return inputs
      }
      return null
    },
    notValid: function() {
      //@todo valid the address
      const gasPriceCheck = /^\d+\.?\d*$/.test(this.gasPrice) && this.gasPrice > 0
      const gasLimitCheck = /^\d+\.?\d*$/.test(this.gasLimit) && this.gasLimit > 0
      const feeCheck = /^\d+\.?\d*$/.test(this.fee) && this.fee > 0.0001
      return !(gasPriceCheck && gasLimitCheck && feeCheck && this.method !== null)
    }
  },
  watch: {
    method: function() {
      this.inputParams = []
    }
  },
  methods: {
    decodeAbi() {
      try {
        const abiJson = JSON.parse(this.abi)
        this.parsedAbi = []
        for (let i = 0; i < abiJson.length; i++) {
          this.parsedAbi[i] = {text: abiJson[i]['name'], value: i, info: abiJson[i]}
        }
      } catch (e) {
        this.$root.log.error('send_to_contract_decode_abi_error', e.stack || e.toString() || e)
        return true
      }
    },
    async send() {
      try {
  const encodedData = abi.encodeMethod(this.parsedAbi[this.method].info, this.inputParams).substr(2)
        this.confirmSendDialog = true
        try {
          this.rawTx = await webWallet.getWallet().generateSendToContractTx(this.contractAddress, encodedData, this.gasLimit, this.gasPrice, this.fee)
        } catch (e) {
          this.$root.log.error('send_to_generate_tx_error', e.stack || e.toString() || e)
          alert(e.message || e)
          this.confirmSendDialog = false
          return false
        }
        this.canSend = true
      } catch (e) {
        this.$root.error('Params error')
  alert(e.message || e)
        this.$root.log.error('send_to_contract_encode_abi_error', e.stack || e.toString() || e)
        this.confirmSendDialog = false
        return false
      }
    },

    async confirmSend() {
      this.sending = true
      try {
        const txId = await webWallet.getWallet().sendRawTx(this.rawTx)
        this.confirmSendDialog = false
        this.sending = false
        const txViewUrl = server.currentNode().getTxExplorerUrl(txId);
        this.$root.success(`Successful sent! You can follow the transaction on <a href="${txViewUrl}" target="_blank">${txViewUrl}</a>`, true, 0);
        this.$emit('send')
      } catch (e) {
        alert(e.message || e)
        this.$root.log.error('send_to_contract_post_raw_tx_error', e.response || e.stack || e.toString() || e)
        this.confirmSendDialog = false
      }
    }
  }
}
</script>
