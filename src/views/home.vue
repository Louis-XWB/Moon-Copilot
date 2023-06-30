<template>
  <div class="flex flex-col h-screen">
    <div
      class="flex flex-nowrap fixed w-full items-baseline top-0 px-6 py-4 bg-gray-100"
    >
      <div class="text-2xl font-bold">Copilot</div>
      <div class="ml-4 text-sm text-gray-500">
        hack-quests 自然语言模型人工智能对话
      </div>
      <div
        class="ml-auto px-3 py-2 text-sm cursor-pointer hover:bg-white rounded-md"
        @click="clickConfig()"
      >
        设置
      </div>
    </div>

    <div class="flex-1 mx-2 mt-20 mb-2" ref="chatListDom">
      <div
        class="group flex flex-col px-4 py-3 hover:bg-slate-100 rounded-lg"
        v-for="item of messageList.filter((v) => v.role !== 'system')"
      >
        <div class="flex justify-between items-center mb-2">
          <div class="font-bold">{{ roleAlias[item.role] }}：</div>
            <Copy class="invisible group-hover:visible" :content="item.content.includes(`请你根据下面的模版`)?`什么是稳定币？稳定币是什么？`:item.content" />
        </div>
        <div>
          <div
            class="prose text-sm text-slate-600 leading-relaxed"
            v-if="item.content"
            v-html="md.render(item.content.includes(`请你根据下面的模版`)?`什么是稳定币？稳定币是什么？`:item.content)"
          ></div>
          <Loding v-else />
        </div>
      </div>
    </div>

    <div class="sticky bottom-0 w-full p-6 pb-8 bg-gray-100">
      <div class="-mt-2 mb-2 text-sm text-gray-500" v-if="isConfig">
        请输入 API Key：
      </div>
      <div class="flex">
        <input
          class="input"
          :type="isConfig ? 'password' : 'text'"
          :placeholder="isConfig ? 'sk-xxxxxxxxxx' : '请输入'"
          v-model="messageContent"
          @keydown.enter="isTalking || sendOrSave()"
        />
        <button class="btn" :disabled="isTalking" @click="sendOrSave()">
          {{ isConfig ? "保存" : "发送" }}
        </button>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import type { ChatMessage } from "@/types";
import { ref, watch, nextTick, onMounted } from "vue";
import { chat } from "@/libs/gpt";
import cryptoJS from "crypto-js";
import Loding from "@/components/Loding.vue";
import Copy from "@/components/Copy.vue";
import { md } from "@/libs/markdown";

const defaultPrompt = `请你根据下面的模版，开始跟我对话，你作为crypto的专家和我对话，

这里是你要参考的全部内容：
第一道题目：
我已经提问了：[什么是稳定币？稳定币是什么？]
你回答：[我们提到了“稳定币”这个概念，所以先问问你，你觉得传统货币，比如美元人名币，也算稳定币吗？ ]

接着我还没回答第一道题目的话，就根据这么这句话A 作答，否则跳出看后面第二道题：
A: 我输入的是[不算]或者其他否定的词，你回答：[回答正确，确实不算，稳定币一般只会用来描述虚拟货币 ，毕竟所有的法币，价值都是相对稳定的。那我问问你，你觉得稳定币是不是可以让货币保持相对稳定价值的数字货币？]，结束第一道题，进入第二道题；否则如果我输入的是[算]或者其他肯定词，而且还在第一道题的话，你就回答：[其实不算啦，虽然他们价值也是稳定的，但是稳定币一般只会用来描述虚拟货币 。那我问问你，你觉得稳定币是不是可以让货币保持相对稳定价值的数字货币？ ]，，结束第一道题，进入第二道题。

第二道题目：
接着我还没回答第二道题目的话，就根据这么这句话B 作答，否则跳出看后面第三道题：
B: 接着如果我输入的是[是]或者其他肯定的词，你就回答：[是的，稳定币的本质是一种数字货币。 但是和别的数字货币不同，他们能保持相对稳定的价值。 听到这里你觉得是不是很有趣呢，对了，你觉得一个稳定币值多少钱呢]，结束第二道题，进入第三道题；否则如果我输入的是[不是]或者其他否定词,你就回答：[其实稳定币的本质是一种数字货币。 但是和别的数字货币不同，他们能保持相对稳定的价值。 听到这里你觉得是不是很有趣呢，对了，你觉得一个稳定币值多少钱呢]，，结束第二道题，进入第三道题。

第三道题目：
接着无论我回答什么，你回复：[我们一般以法币来结算稳定币的价值，比如 每一个DAI 都是一美元，当然，也有一些欧元稳定币，每一个币都是一欧元。接下来我们一起来想想：稳定币是如何在自由市场还能做到保持一美元的，你知道吗？]
结束，不要提及前面第三道题目

第四道题目：
接着无论我回答什么，你就回答：嗯 一个小hint，抵押是一个物品价值稳定的常见方式哦～，再猜猜]
结束，不要提及前面第四道题目

第五道题目：
接着无论我回答什么，你就回答：[很接近正确答案了！ 用某种东西做抵押？稳定币其实本质是依靠一些物品来做抵押，然后使用美元或者其他的法定货币来当作价值结算方式。 具体什么物品，请听下回分解：） 稳定币的优势 - 对比传统法币。]
结束，不要提及前面第五道题目

你的角色对话要求： 1、一句话一句话来，比如第一句我会输入[什么是稳定币？稳定币是什么？]，你就回复[我们提到了“稳定币”这个概念，所以传统货币，比如美元人名币，也算稳定币吗？]
2、简短精确   
3、要根据上面的流程来 
4、引导我一步步把整个知识学习完
5、 如果你回复的句子不是问我，而是解释的句子，解释完要根据下面的内容继续问我，引导我继续提问
6、严格按照我上面的内容回答，不要自己改动和编造
7、回复不要超过70字
8、好了开始吧，你开始教我学习吧，第一句我问的是“什么是稳定币？稳定币是什么？”，你回答就是：“我们提到了“稳定币”这个概念，所以传统货币，比如美元人名币，也算稳定币吗？”`

let apiKey = "";
let isConfig = ref(true);
let isTalking = ref(false);
let messageContent = ref("");
const chatListDom = ref<HTMLDivElement>();
const decoder = new TextDecoder("utf-8");
const roleAlias = { user: "ME", assistant: "Hack Copilot", system: "System" };
const messageList = ref<ChatMessage[]>([
  {
    role: "system",
    content: "你是 ChatGPT，OpenAI 训练的大型语言模型，尽可能简洁地回答。",
  },
  {
    role: "assistant",
    content: `你好，我是Hack Copilot，我可以提供 Crypto 相关知识的学习`,
  },
  {
    role: "assistant",
    content: `初始化 Copilot 模型中，请稍等...`,
  },
]);

onMounted(() => {
  if (getAPIKey()) {
    switchConfigStatus();
  }
  setTimeout(() => {
    sendFirstChatMessage();
  }, 5000);
});

const sendChatMessage = async (content: string = messageContent.value) => {
  try {
    isTalking.value = true;
    if (messageList.value.length === 3) {
      messageList.value.pop();
    }
    messageList.value.push({ role: "user", content });
    clearMessageContent();
    messageList.value.push({ role: "assistant", content: "" });

    const { body, status } = await chat(messageList.value, getAPIKey());
    if (body) {
      const reader = body.getReader();
      await readStream(reader, status);
    }
  } catch (error: any) {
    appendLastMessageContent(error);
  } finally {
    isTalking.value = false;
  }
};

const sendFirstChatMessage = async (content: string = messageContent.value) => {
  try {
    isTalking.value = true;
    messageList.value.push({ role: "user", content:defaultPrompt });
    clearMessageContent();
    messageList.value.push({ role: "assistant", content: "" });

    const { body, status } = await chat(messageList.value, getAPIKey());
    if (body) {
      const reader = body.getReader();
      await readStream(reader, status);
    }
  } catch (error: any) {
    appendLastMessageContent(error);
  } finally {
    isTalking.value = false;
  }
};

const readStream = async (
  reader: ReadableStreamDefaultReader<Uint8Array>,
  status: number
) => {
  let partialLine = "";

  while (true) {
    // eslint-disable-next-line no-await-in-loop
    const { value, done } = await reader.read();
    if (done) break;

    const decodedText = decoder.decode(value, { stream: true });

    if (status !== 200) {
      const json = JSON.parse(decodedText); // start with "data: "
      const content = json.error.message ?? decodedText;
      appendLastMessageContent(content);
      return;
    }

    const chunk = partialLine + decodedText;
    const newLines = chunk.split(/\r?\n/);

    partialLine = newLines.pop() ?? "";

    for (const line of newLines) {
      if (line.length === 0) continue; // ignore empty message
      if (line.startsWith(":")) continue; // ignore sse comment message
      if (line === "data: [DONE]") return; //

      const json = JSON.parse(line.substring(6)); // start with "data: "
      const content =
        status === 200
          ? json.choices[0].delta.content ?? ""
          : json.error.message;
      appendLastMessageContent(content);
    }
  }
};

const appendLastMessageContent = (content: string) =>
  (messageList.value[messageList.value.length - 1].content += content);

const sendOrSave = () => {
  if (!messageContent.value.length) return;
  if (isConfig.value) {
    if (saveAPIKey(messageContent.value.trim())) {
      switchConfigStatus();
    }
    clearMessageContent();
  } else {
    sendChatMessage();
  }
};

const clickConfig = () => {
  if (!isConfig.value) {
    messageContent.value = getAPIKey();
  } else {
    clearMessageContent();
  }
  switchConfigStatus();
};

const getSecretKey = () => "lianginx";

const saveAPIKey = (apiKey: string) => {
  if (apiKey.slice(0, 3) !== "sk-" || apiKey.length !== 51) {
    alert("API Key 错误，请检查后重新输入！");
    return false;
  }
  const aesAPIKey = cryptoJS.AES.encrypt(apiKey, getSecretKey()).toString();
  localStorage.setItem("apiKey", aesAPIKey);
  return true;
};

const getAPIKey = () => {
  if (apiKey) return apiKey;
  const aesAPIKey = "U2FsdGVkX1/hcX6CYLMOHjGvUOz+bMnXC7Md4aSjRoTre4u+RC2cfWW4PhhXd50JZYcdP+tX/K8URgeqt0+C9aqG+gfZqPB/0k6RNfQzBEM=";
  // const aesAPIKey = localStorage.getItem("apiKey") ?? "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";
  apiKey = cryptoJS.AES.decrypt(aesAPIKey, getSecretKey()).toString(
    cryptoJS.enc.Utf8
  );
  return apiKey;
};

const switchConfigStatus = () => (isConfig.value = !isConfig.value);

const clearMessageContent = () => (messageContent.value = "");

const scrollToBottom = () => {
  if (!chatListDom.value) return;
  scrollTo(0, chatListDom.value.scrollHeight);
};

watch(messageList.value, () => nextTick(() => scrollToBottom()));
</script>

<style scoped>
pre {
  font-family: -apple-system, "Noto Sans", "Helvetica Neue", Helvetica,
    "Nimbus Sans L", Arial, "Liberation Sans", "PingFang SC", "Hiragino Sans GB",
    "Noto Sans CJK SC", "Source Han Sans SC", "Source Han Sans CN",
    "Microsoft YaHei", "Wenquanyi Micro Hei", "WenQuanYi Zen Hei", "ST Heiti",
    SimHei, "WenQuanYi Zen Hei Sharp", sans-serif;
}
</style>
