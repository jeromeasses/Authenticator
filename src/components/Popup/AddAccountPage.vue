<template>
  <div>
    <label>{{ i18n.issuer }}</label>
    <input type="text" class="input" v-model="newAccount.issuer" />
    <label>{{ i18n.secret }}</label>
    <input type="text" class="input" v-model="newAccount.secret" />
    <details>
      <summary>{{ i18n.advanced }}</summary>
      <label>{{ i18n.accountName }}</label>
      <input type="text" class="input" v-model="newAccount.account" />
      <label>{{ i18n.period }}</label>
      <input
        type="number"
        min="1"
        class="input"
        v-model.number="newAccount.period"
        v-bind:disabled="newAccount.type === OTPType.hotp"
      />
      <div class="control-group">
        <label class="combo-label" style="margin: 20px 10px;">{{
          i18n.digits
        }}</label>
        <select style="margin: 20px 10px;" v-model="newAccount.digits">
          <option value="6">6</option>
          <option value="8">8</option>
        </select>
      </div>
      <div class="control-group">
        <label class="combo-label" style="margin: 20px 10px;">{{
          i18n.algorithm
        }}</label>
        <select style="margin: 20px 10px;" v-model="newAccount.algorithm">
          <option v-bind:value="OTPAlgorithm.SHA1">SHA-1</option>
          <option v-bind:value="OTPAlgorithm.SHA256">SHA-256</option>
          <option v-bind:value="OTPAlgorithm.SHA512">SHA-512</option>
        </select>
      </div>
      <div class="control-group">
        <label class="combo-label" style="margin: 20px 10px;">{{
          i18n.type
        }}</label>
        <select style="margin: 20px 10px;" v-model="newAccount.type">
          <option v-bind:value="OTPType.totp">{{ i18n.based_on_time }}</option>
          <option v-bind:value="OTPType.hotp">{{
            i18n.based_on_counter
          }}</option>
          <option v-bind:value="OTPType.battle">Battle.net</option>
          <option v-bind:value="OTPType.steam">Steam</option>
        </select>
      </div>
    </details>
    <div class="button-small" v-on:click="addNewAccount()">{{ i18n.ok }}</div>
  </div>
</template>
<script lang="ts">
import Vue from "vue";
import { mapState } from "vuex";
import { OTPType, OTPEntry, OTPAlgorithm } from "../../models/otp";

export default Vue.extend({
  data: function(): {
    newAccount: {
      issuer: string;
      account: string;
      secret: string;
      type: OTPType;
      period: number | undefined;
      digits: number;
      algorithm: OTPAlgorithm;
    };
  } {
    return {
      newAccount: {
        issuer: "",
        account: "",
        secret: "",
        type: OTPType.totp,
        period: undefined,
        digits: 6,
        algorithm: OTPAlgorithm.SHA1
      }
    };
  },
  computed: mapState("accounts", ["OTPType", "OTPAlgorithm"]),
  methods: {
    async addNewAccount() {
      this.newAccount.secret = this.newAccount.secret.replace(/ /g, "");

      if (this.newAccount.secret.length < 16) {
        this.$store.commit("notification/alert", this.i18n.errorsecret);
        return;
      }

      if (
        !/^[a-z2-7]+=*$/i.test(this.newAccount.secret) &&
        !/^[0-9a-f]+$/i.test(this.newAccount.secret)
      ) {
        this.$store.commit("notification/alert", this.i18n.errorsecret);
        return;
      }
      let type: OTPType;
      if (
        !/^[a-z2-7]+=*$/i.test(this.newAccount.secret) &&
        /^[0-9a-f]+$/i.test(this.newAccount.secret) &&
        this.newAccount.type === OTPType.totp
      ) {
        type = OTPType.hex;
      } else if (
        !/^[a-z2-7]+=*$/i.test(this.newAccount.secret) &&
        /^[0-9a-f]+$/i.test(this.newAccount.secret) &&
        this.newAccount.type === OTPType.hotp
      ) {
        type = OTPType.hhex;
      } else {
        type = this.newAccount.type;
      }

      if (type === OTPType.hhex || type === OTPType.hotp) {
        this.newAccount.period = undefined;
      } else if (
        typeof this.newAccount.period !== "number" ||
        this.newAccount.period < 1
      ) {
        this.newAccount.period = undefined;
      }

      const entry = new OTPEntry(
        {
          type,
          index: 0,
          issuer: this.newAccount.issuer,
          account: this.newAccount.account,
          encrypted: false,
          secret: this.newAccount.secret,
          counter: 0,
          period: this.newAccount.period,
          digits: this.newAccount.digits,
          algorithm: this.newAccount.algorithm
        },
        this.$store.state.accounts.encryption
      );

      await entry.create();
      await this.$store.dispatch("accounts/addCode", entry);
      this.$store.commit("style/hideInfo");
      this.$store.commit("style/toggleEdit");

      const codes = document.getElementById("codes");
      if (codes) {
        // wait vue apply changes to dom
        setTimeout(() => {
          codes.scrollTop = 0;
        }, 0);
      }
      return;
    }
  }
});
</script>
