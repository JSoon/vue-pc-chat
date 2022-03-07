<template>
  <div class="login-hailuo">
    <form @submit.prevent="onLoginSubmit">
      <div>
        <label for="loginName">用户名</label>
        <input id="loginName" type="text" name="account" value="" />
      </div>
      <div>
        <label for="loginPwd">密码</label>
        <input id="loginPwd" type="password" name="password" value="" />
      </div>
      <div>
        <button type="submit">登录</button>
      </div>
    </form>
  </div>
</template>

<script>
import wfc from "../../wfc/client/wfc";
import axios from "axios";
import JSEncrypt from "jsencrypt";
import { setItem } from "../util/storageHelper";
import ConnectionStatus from "../../wfc/client/connectionStatus";
import EventType from "../../wfc/client/wfcEvent";

axios.defaults.baseURL = "https://hlapitest.jwell56.com";
axios.defaults.headers.common.client = "hlweb";

// 密码加密key
const PUBLIC_KEY =
  "MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQCqqvBIBvv53+8M8CfJx6hdbTr5quw95p+4UgzBYHYbHayl8TjlgtgzZ9tmxSK1BkdB7hnQLxv3zCiItbyVcJGCGj1uuyP9iZeKZZVgs7/T4J8u+eWVIXvNIFqAT/+o5qctC7FZ3Vs1sfjA/30lQ3q3EyF7UgaHJhdyvtyY7QpQ8QIDAQAB";

const encrptPwd = originPwd => {
  const encrypt = new JSEncrypt();
  encrypt.setPublicKey(PUBLIC_KEY);
  return encrypt.encrypt(originPwd);
};

export default {
  created() {
    wfc.eventEmitter.on(
      EventType.ConnectionStatusChanged,
      this.onConnectionStatusChange
    );
  },
  beforeDestroy() {
    wfc.eventEmitter.removeListener(
      EventType.ConnectionStatusChanged,
      this.onConnectionStatusChange
    );
  },
  methods: {
    // 登录提交
    async onLoginSubmit(e) {
      const formData = new FormData(e.target);
      const formProps = Object.fromEntries(formData);

      // 1. 获取access token
      const { data: loginRes } = await axios.post(
        "/jplat/hlweb/login/passwordLogin",
        null,
        {
          params: {
            account: formProps.account,
            password: encrptPwd(formProps.password)
          }
        }
      );
      const token = loginRes.data.accessToken;
      axios.defaults.headers.common["Authorization"] = `bearer ${token}`;

      // 2. 获取用户信息
      const { data: userInfoRes } = await axios.get(
        "/jplat/hlweb/getUserByToken"
      );
      const userId = userInfoRes.data.id;
      axios.defaults.headers.common["id"] = userId || "";
      axios.defaults.headers.common["accountid"] =
        userInfoRes.data.accountid || "";

      // 3. 获取im token
      const { data: imRes } = await axios.post("/imserver/im/getToken", null, {
        params: {
          userId,
          clientId: wfc.getClientId(),
          platform: 5
        }
      });
      const imToken = imRes.data.token;

      setItem("userId", userId);
      setItem("token", token);
      setItem("imtoken", imToken);

      wfc.connect(userId, imToken);
    },
    // IM连接状态变化事件
    onConnectionStatusChange(status) {
      if (status === ConnectionStatus.ConnectionStatusConnected) {
        this.$router.replace({ path: "/home" });
      }
    }
  }
};
</script>

<style></style>
