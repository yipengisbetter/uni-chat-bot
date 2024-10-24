<template>
  <view class="page">
    <view class="header">
      <view class="btn">
        <image class="img" src="../../static/back.svg"></image>
      </view>
      <view class="info">
        <view class="title">新的聊天</view>
        <view class="count">共{{ messageList.length }}条对话</view>
      </view>
      <view class="btn">
        <image class="img" src="../../static/share.svg"></image>
      </view>
    </view>
    <view class="content">
      <view class="message-list">
        <view v-for="message of renderMessageList" :class="{ message: true, user: message.role === 'user' }">
          <rich-text class="richText" :nodes="message.content"></rich-text>
        </view>
      </view>
    </view>
    <view class="footer">
      <view class="input-container">
        <textarea v-model="userInput" class="textarea" placeholder="请输入您的问题" />
        <view class="send-btn" @click="send()">
          <image class="img" src="../../static/send.svg"></image>
          <view class="txt">发送</view>
        </view>
      </view>
    </view>
  </view>
</template>

<script lang="ts">
import Vue from 'vue'
import { fetchEventSource } from '@microsoft/fetch-event-source'
import * as TextEncoding from 'text-encoding-shim'
import { marked } from 'marked'

type Message = {
  role: "user" | "assistant"
  content: string
}
declare const wx: any

export default Vue.extend({
  data() {
    return {
      messageList: new Array<Message>(),
      userInput: ""
    }
  },
  computed: {
    renderMessageList() {
      return (this as any).messageList.map((i: any) => ({ content: marked.parse(i.content), role: i.role }))
    }
  },
  onLoad() {
    this.messageList = JSON.parse(uni.getStorageSync("messageList"))
  },
  methods: {
    async send() {
      this.messageList = [...this.messageList, { role: "user", content: this.userInput }]
      this.userInput = ""

      let replyText = ""

      const messages = JSON.parse(JSON.stringify(this.messageList))
      this.messageList.push({
        role: "assistant",
        content: replyText
      })
      const lastMessage = this.messageList[this.messageList.length - 1]

      const systemInfo = await uni.getSystemInfo()
      if (systemInfo.uniPlatform == "web") {
        await fetchEventSource("https://aip.baidubce.com/rpc/2.0/ai_custom/v1/wenxinworkshop/chat/ernie-lite-8k?access_token=24.336aea537d19a1ca4620dca2905ca2b3.2592000.1731853048.282335-115921357", {
          method: "POST",
          headers: {
            'Content-Type': 'application/json'
          },
          body: JSON.stringify({ messages, stream: true }),
          onmessage(ev) {
            replyText += JSON.parse(ev.data).result
            lastMessage.content = replyText
          }
        });
        uni.setStorage({ key: "messageList", data: JSON.stringify(this.messageList) })
      } else if (systemInfo.uniPlatform == "mp-weixin") {

        const requestTask = wx.request({
          url: "https://aip.baidubce.com/rpc/2.0/ai_custom/v1/wenxinworkshop/chat/ernie-lite-8k?access_token=24.336aea537d19a1ca4620dca2905ca2b3.2592000.1731853048.282335-115921357",
          method: "POST",
          enableChunked: true,
          data: {
            messages,
            stream: true
          },
          success: (response: any) => {
            // 开启enableChunked后，成功的回调一般用不到，因为响应数据不在这里返回
            console.log(response)
          }
        })

        // 这里是注释
        requestTask.onChunkReceived((chunk: any) => {
          const arrayBuffer = chunk.data;
          const uint8Array = new Uint8Array(arrayBuffer);
          const str = new TextEncoding.TextDecoder('utf-8').decode(uint8Array);
          str.split(/\n\n/).forEach(i => {
            if (i) {
              // const data = i.replace("data: ", "")
              const data = i.slice(6)
              replyText += JSON.parse(data).result
              lastMessage.content = replyText
            }
          })
          // 看一下，打印出来的结果
          console.log(str)
        })
      }

      // const res = await uni.request({
      //   url: "https://aip.baidubce.com/rpc/2.0/ai_custom/v1/wenxinworkshop/chat/ernie-lite-8k?access_token=24.336aea537d19a1ca4620dca2905ca2b3.2592000.1731853048.282335-115921357",
      //   method: "POST",
      //   data: { messages: this.messageList, stream: true }
      // })
      // this.messageList.push({
      //   role: "assistant",
      //   content: (res.data as any).result
      // })
    }
  }
});
</script>

<style scoped>
.header {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 14px 20px;
  border-bottom: 1px solid #dedede;
  background-color: #fff;
  z-index: 1;
}

.btn {
  border: 1px solid #dedede;
  padding: 16px;
  border-radius: 10px;
  height: 16px;
}

.img {
  width: 16px;
  height: 16px;
}

.info {
  text-align: center;
}

.title {
  font-size: 20px;
  font-weight: 700;
}

.count {
  font-size: 14px;
}

.footer {
  background-color: #fff;
  position: fixed;
  bottom: 0;
  left: 0;
  right: 0;
  padding: 20px;
  border-top: 1px solid #dedede;
}

.input-container {

  display: flex;
  border: 1px solid #dedede;
  padding: 10px 14px;
  border-radius: 10px;
}

.textarea {
  flex: 1;
  height: 46px;
}

.send-btn {
  background-color: #1d93ab;
  color: #fff;
  display: flex;
  padding: 16px;
  font-size: 12px;
  border-radius: 10px;
}

.txt {
  margin-left: 4px;
}

.content {
  padding: 78px 0 110px;
}

.message-list {
  padding: 20px;
  display: flex;
  flex-direction: column;
  align-items: flex-start;
}

.message {
  font-size: 14px;
  color: #24292f;
  padding: 10px;
  background-color: rgba(0, 0, 0, .05);
  border-radius: 10px;
  border: 1px solid #dedede;
  margin-bottom: 26px;
}

.message.user {
  align-self: flex-end;
  background-color: #e7f8ff;
}
</style>