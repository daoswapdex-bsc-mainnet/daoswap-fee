<template>
  <div class="fill-height">
    <v-container v-if="web3 && connected" class="fill-height">
      <v-row justify="center">
        <v-col md="6">
          <v-card class="fill-width">
            <v-card-title>
              <v-avatar size="24" class="mr-2">
                <img :src="require('@/assets/logo.png')" alt="list" />
              </v-avatar>
              <span class="title font-weight-light">
                {{ $t("LP List") }}
              </span>
            </v-card-title>
            <v-card-text v-if="chainId === 56 && rewardsDataList.length > 0">
              <v-card
                v-for="item in rewardsDataList"
                :key="item.address"
                :loading="loading"
                class="mb-4"
              >
                <v-card-title>
                  <span class="title font-weight-light">
                    {{ item.name }} {{ $t("Available Amount") }}
                  </span>
                </v-card-title>
                <v-divider class="mx-4"></v-divider>
                <v-card-text>
                  <v-row align="center">
                    <v-col class="display-3" cols="12">
                      {{ item.rewardsAmount }}
                    </v-col>
                  </v-row>
                </v-card-text>
                <v-divider
                  class="mx-4"
                  v-if="parseFloat(item.rewardsAmount) > 0"
                ></v-divider>
                <v-card-actions
                  class="justify-center"
                  v-if="parseFloat(item.rewardsAmount) > 0"
                >
                  <v-btn
                    color="#93B954"
                    dark
                    width="80%"
                    @click="submit(item)"
                    :disabled="parseFloat(item.rewardsAmount) <= 0"
                  >
                    {{ $t("Claim") }}
                  </v-btn>
                </v-card-actions>
                <v-divider class="mx-4"></v-divider>
                <v-card-text>
                  <v-row align="center">
                    <v-col class="body-1" cols="12">
                      <p @click="handleCopy(item.lpAddress, $event)">
                        LP Address: {{ item.lpAddress }}
                        <v-icon>mdi-content-copy</v-icon>
                      </p>
                    </v-col>
                  </v-row>
                </v-card-text>
              </v-card>
            </v-card-text>
            <v-card-text v-else>
              <v-row align="center">
                <v-col class="body-3" cols="12">
                  {{ $t("No Data") }}
                </v-col>
              </v-row>
            </v-card-text>
          </v-card>
          <!-- 当前钱包账号 -->
          <v-card justify="center" class="fill-width mt-10">
            <v-card-title>
              <span class="title font-weight-light">
                {{ $t("Current Token Address") }}
              </span>
            </v-card-title>
            <v-divider></v-divider>
            <v-card-text>
              <v-row align="center">
                <v-col
                  class="body-1"
                  cols="12"
                  @click="handleCopy(address, $event)"
                >
                  <p>
                    {{ address }}
                    <v-icon>mdi-content-copy</v-icon>
                  </p>
                </v-col>
              </v-row>
            </v-card-text>
          </v-card>
          <!-- 遮罩层 -->
          <v-overlay z-index="9999" opacity="0.7" :value="loading">
            <v-progress-circular indeterminate size="64"></v-progress-circular>
          </v-overlay>
          <!-- 提示层 -->
          <v-snackbar
            v-model="operationResult.snackbar"
            :color="operationResult.color"
            centered
            top
            timeout="4000"
          >
            {{ $t(operationResult.text) }}
          </v-snackbar>
        </v-col>
      </v-row>
    </v-container>
    <v-container v-else>
      <v-row justify="center">
        <v-col md="6">
          <v-card justify="center" class="fill-width">
            <v-card-actions class="justify-center">
              <!-- 连接钱包 -->
              <v-btn
                class="mr-2"
                v-if="!connected"
                color="#93B954"
                block
                @click="onConnect"
              >
                {{ $t("Connect Wallet") }}
              </v-btn>
            </v-card-actions>
          </v-card>
        </v-col>
      </v-row>
    </v-container>
  </div>
</template>

<script>
import { validationMixin } from "vuelidate";
import clip from "@/utils/clipboard";
import { compare } from "@/filters/index";
import { getContractByABI, weiToEther, etherToWei } from "@/utils/web3";
// 引入合约 ABI 文件
import CommunityRewards_ABI from "@/constants/contractJson/CommunityRewardsApprove_abi.json";

export default {
  name: "AllocationFee",
  mixins: [validationMixin],
  data: () => ({
    loading: false,
    // LP合约列表
    lpContractList: [
      {
        name: "USDT-ETH",
        address: "0xd98229FF432DBcc32B2805F9D3dEaF8DAc5f864b",
        lpAddress: "0x4daa66817F5436ed15879A3E1eBD6a5fd510131F"
      },
      {
        name: "USDT-DAO",
        address: "0x94Dc552A6d25585DAE7093681f1C33C3FE8837D5",
        lpAddress: "0x69217cF12bD760D6eA850A128646b21F7A61EE07"
      },
      {
        name: "USDT-BTC",
        address: "0x37e51f00B3F55895034c328eAa6630e328d22cc1",
        lpAddress: "0x3a5A8B0363717C97Fa541b51Eb3C4e062E309A9C"
      },
      {
        name: "USDT-FIL",
        address: "0x3fa1F2E69D43C0A4ABb034F445f6b878F866A3Fb",
        lpAddress: "0xd952c06Cd22361ce5538d357B8F4F09f6F198641"
      }
    ],
    // 记录列表
    rewardsDataList: [],
    // 提示框
    operationResult: {
      color: "success",
      snackbar: false,
      text: `Hello`
    }
  }),
  created() {
    if (this.web3 && this.connected) {
      this.getAccountAssets();
    } else {
      this.onConnect();
    }
  },
  watch: {
    web3(web3) {
      if (web3) {
        this.getAccountAssets();
      }
    },
    address(address) {
      if (address) {
        this.getAccountAssets();
      }
    }
  },
  computed: {
    connected() {
      return this.$store.state.web3.connected;
    },
    web3() {
      return this.$store.state.web3.web3;
    },
    address() {
      return this.$store.state.web3.address;
    },
    chainId() {
      return this.$store.state.web3.chainId;
    }
  },
  methods: {
    // 连接钱包 OK
    onConnect() {
      this.$store.dispatch("web3/connect");
    },
    // 断开连接钱包 OK
    closeWallet() {
      this.$store.dispatch("web3/closeWallet");
    },
    // 复制地址
    handleCopy(text, event) {
      clip(text, event);
      this.operationResult.color = "success";
      this.operationResult.snackbar = true;
      this.operationResult.text = "Cope Success";
    },
    // 获取账号信息
    async getAccountAssets() {
      this.loading = true;
      try {
        if (this.lpContractList.length > 0) {
          this.rewardsDataList = [];
          this.loading = true;
          const getResult = this.lpContractList.map(async item => {
            const contract = await getContractByABI(
              CommunityRewards_ABI,
              item.address,
              this.web3
            );
            const rewardsInfo = await contract.methods
              .getRewardsInfoByAccount()
              .call({ from: this.address });
            const rewardsAmount = weiToEther(
              rewardsInfo.rewardsAmount,
              this.web3
            );
            if (parseFloat(rewardsAmount) >= 0) {
              const tempData = {
                name: item.name,
                address: item.address,
                lpAddress: item.lpAddress,
                rewardsAmount: rewardsAmount
              };
              this.rewardsDataList.push(tempData);
            }
          });
          await Promise.all(getResult);
          this.rewardsDataList.sort(compare("address"));
          this.loading = false;
        }
      } catch (error) {
        console.info(error);
      }
      this.loading = false;
    },
    // 提币 TODO OK
    submit(record) {
      this.loading = true;
      // 处理额度
      const rewardsAmount = etherToWei(record.rewardsAmount, this.web3);
      // 执行合约
      getContractByABI(CommunityRewards_ABI, record.address, this.web3)
        .methods.getReward(rewardsAmount)
        .send({ from: this.address })
        .then(() => {
          this.loading = false;
          this.getAccountAssets();
        })
        .catch(e => {
          this.loading = false;
          console.info(e);
        });
    }
  }
};
</script>

<style lang="sass">
.theme--dark.v-btn.v-btn--disabled.v-btn--has-bg
  background-color: rgb(147, 185, 84) !important
  border-color: rgb(147, 185, 84) !important
  opacity: 0.5 !important

.v-btn--disabled
  background-color: rgb(147, 185, 84)
  border-color: rgb(147, 185, 84)
  opacity: 0.5
</style>
