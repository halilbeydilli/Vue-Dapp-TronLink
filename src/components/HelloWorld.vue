<script setup>
import { ref } from 'vue';
const { tronWeb } = window;

const account = ref(null);
const nativeBalance = ref(null);
const signedMessage = ref(null);


const connectWallet = async () => {
  try {
    if (account.value) {
      account.value = null;
      signedMessage.value = null;
      return await tronWeb.disconnect();
    }
    if (!tronWeb) {
      return console.log('Install Tronlink first');
    }

    let address = tronWeb.defaultAddress.base58;
    if (!address) {
      await tronWeb.request(
        {
          method: 'tron_requestAccounts',
          params: {
            websiteIcon: '',
            websiteName: 'Test Dapps',
          },
        }
      )
      address = tronWeb.defaultAddress.base58;
    }
    if (address) {
      account.value = address;
      await checkNativeBalance(address);
    }
  } catch (error) {
    console.log(error.message)
  }

}

const checkNativeBalance = async (address) => {
  try {
    const trxBalance = await tronWeb.trx.getBalance(address);
    nativeBalance.value = trxBalance / 10 ** 6;
  } catch (error) {
    console.log(error.message)
  }
}

const signMessage = async () => {
  try {
    const message = 'Sign this test message';
    const hexStrWithout0x = await tronWeb.toHex(message).replace(/^0x/, "");
    const byteArray = await tronWeb.utils.code.hexStr2byteArray(hexStrWithout0x);
    const strHash = tronWeb.sha3(byteArray).replace(/^0x/, "");
    let signedStr = await tronWeb.trx.sign(strHash);
    const tail = signedStr.substring(128, 130);
    if (tail == "01") {
      signedStr = (await signedStr.substring(0, 128)) + "1c";
    } else if (tail == "00") {
      signedStr = (await signedStr.substring(0, 128)) + "1b";
    }
    const signedMessageStatus = await tronWeb.trx.verifyMessage(strHash, signedStr, account.value);
    if (signedMessageStatus) {
      signedMessage.value = signedStr;
    }
  } catch (error) {
    console.log(error)
  }
}

const sendTransaction = async () => {
  try {
    const amount = 1 * 10 ** 6;
    const account = tronWeb.defaultAddress.base58;
    const toAddress = 'TWLA3eauH8q6TcYLqCfh8eFRoeP7s1hyGm'
    const tx = await tronWeb.transactionBuilder.sendTrx(await tronWeb.address.fromHex(toAddress), amount);
    const signedTx = await tronWeb.trx.sign(tx);
    const hash = await tronWeb.trx.sendRawTransaction(signedTx);
    console.log(hash.txid)
  } catch (error) {
    console.log(error.message)
  }
}
</script>

<template>
  <div class="greetings">
    <div v-if="account"> Address: {{ account }} </div>
    <div v-if="nativeBalance && account"> Native Balance: {{ nativeBalance }} </div>
    <div v-if="signedMessage && account"> Sign Message: {{ signedMessage }} </div>
    <div class="connectWallet">
      <button v-if="account" class="button" @click="signMessage()">Sign Message</button>
      <button v-if="account" class="button" @click="sendTransaction()">Send Transaction</button>
      <button class="button" @click="connectWallet()">{{ account ? 'Disconnect' : 'Connect Wallet' }}</button>
    </div>
  </div>
</template>

<style scoped>
.address {
  margin-bottom: 30px
}

.connectWallet {
  margin-top: 30px
}

.button {
  margin-right: 5px;
}
</style>
