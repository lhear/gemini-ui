<template>
  <div class="chat-container" ref="chatContainer" @scroll="handleScroll">
    <div v-for="e in slicedContext" class="message-row" :class="{ 'user-message-row': e.role == 'user' }" :key="e.id">
      <div v-for="e1 in e.parts" class="message-bubble"
        :class="{ 'user-bubble': e.role == 'user', 'assistant-bubble': e.role != 'user' }">
        <template v-if="e.role == 'user'">{{ e1.text }}</template>
        <template v-else>
          <div v-html="markdown.render(e1.text)" class="markdown-body"></div>
        </template>
      </div>
    </div>
    <div v-show="showMenu" class="message-bubble menu-bubble">
      <div class="menu-item" @click="showMenu = false; showSetting = true;">设置</div><br><br>
      <div class="menu-item" @click="showMenu = false; removeLast()">删除最后</div><br><br>
      <div class="menu-item" @click="showMenu = false; clear()">清除对话</div>
    </div>
    <div v-show="showSetting" class="message-bubble menu-bubble">
      <div class="menu-item" @click="showSetting = false; setBaseUrl()">设置API地址</div><br><br>
      <div class="menu-item" @click="showSetting = false; setApiKey()">设置API密钥</div><br><br>
      <div class="menu-item" @click="showSetting = false; setModel()">设置模型</div><br><br>
      <div class="menu-item" @click="showSetting = false; setSystemInstruction()">设置系统提示词</div>
      <br><br>
      <div class="menu-item" @click="showSetting = false; setMaxOutputTokens()">设置输出令牌限制</div>
      <br><br>
      <div class="menu-item" @click="showSetting = false; setTemperature()">设置温度</div><br><br>
      <div class="menu-item" @click="showSetting = false; switchSearchDisabled()">{{ searchDisabled ? "启用" : "禁用"
      }}搜索
      </div>
    </div>
    <div v-show="alert.show" class="message-bubble menu-bubble">
      <div class="alert-text">{{ alert.text }}</div><br>
      <div class="menu-item" @click="alert.show = false; alert.onclick()">确定</div>
    </div>
    <div v-show="prompt.show" class="message-bubble menu-bubble">
      <div class="prompt-title">{{ prompt.title }}</div><br>
      <div>
        <input class="prompt-input" placeholder="在此输入" type="text" :value="prompt.text"
          @input="(e) => { prompt.text = e.target.value }" size="25">
      </div><br>
      <div class="menu-item" @click="prompt.show = false; prompt.onclick()">确定</div>
    </div>
  </div>
  <div class="toolbar" ref="toolbar">
    <button @click="clickMenu" class="help-button" :disabled="submitDisabled">{{
      submitDisabled ?
        loadingChats[loadingChatIndex] : "?" }}</button>
    <textarea ref="textarea" v-model="input" placeholder="想说什么呢？" class="input-box"></textarea>
    <button @click="submit" class="submit-button" v-show="!submitDisabled && input.trim().length > 0">
      {{ submitDisabled ? loadingChats[loadingChatIndex] : "⇧" }}
    </button>
  </div>
  <div class="scrollbar-container" ref="scrollbarContainer">
    <div ref="scrollBar" class="scrollbar-thumb"></div>
    <Transition>
      <div class="scrollbar-track-bg" v-show="showScrollBar"></div>
    </Transition>
  </div>
</template>

<script setup>
import { ref, onMounted, watch, nextTick, computed } from "vue";
import { GoogleGenerativeAI } from "@google/generative-ai";
import MarkdownIt from "markdown-it";
import { compress, decompress } from 'lz-string';
import { throttle } from 'lodash-es';

const markdown = new MarkdownIt();
const chatContainer = ref(null);
const input = ref('');
const context = ref([]);
const showMenu = ref(false);
const showSetting = ref(false);
const baseUrl = ref(null);
const apiKey = ref(null);
const aiModel = ref(null);
const systemInstruction = ref(null);
const submitDisabled = ref(false);
const loadingChatIndex = ref(0);
const maxOutputTokens = ref(0);
const temperature = ref(0);
const searchDisabled = ref(false);
const textarea = ref(null);
const toolbar = ref(null);
const renderedContextLength = ref(defaultRenderedContextStep);
const scrollBar = ref(0);
const showScrollBar = ref(false);
const scrollbarContainer = ref(null);

const alert = ref({
  show: false,
  text: '',
  onclick: null
});
const prompt = ref({
  show: false,
  title: '',
  text: '',
  onclick: null
});

let genAI, model, chat, loadingChatInterval,
  handleScrollLock = false, scrollbarHideTimer;

const defaultBaseUrl = "https://generativelanguage.googleapis.com";
const defaultModel = "gemini-2.0-flash";
const defaultSystemInstruction = "总是使用中文回答";
const defaultTemperature = 1.0;
const defaultMaxOutputTokens = 8192;
const defaultSearchDisabled = false;
const defaultRenderedContextStep = 30;

const loadingChats = ["|", "/", "--", "\\"];

const slicedContext = computed(() => {
  return context.value.slice(renderedContextLength.value * -1);
})

onMounted(() => {
  apiKey.value = localStorage.getItem("apiKey");
  baseUrl.value = localStorage.getItem("baseUrl");
  aiModel.value = localStorage.getItem("model");
  systemInstruction.value = localStorage.getItem("systemInstruction")
  let history = localStorage.getItem("history");
  maxOutputTokens.value = localStorage.getItem("maxOutputTokens");
  temperature.value = localStorage.getItem("temperature");
  searchDisabled.value = localStorage.getItem("searchDisabled") == "true" ? true : defaultSearchDisabled;

  if (!baseUrl.value) {
    localStorage.setItem("baseUrl", defaultBaseUrl);
    baseUrl.value = defaultBaseUrl;
  }
  if (!aiModel.value) {
    localStorage.setItem("model", defaultModel);
    aiModel.value = defaultModel;
  }
  if (!systemInstruction.value) {
    localStorage.setItem("systemInstruction", defaultSystemInstruction);
    systemInstruction.value = defaultSystemInstruction;
  }
  let h;
  if (history && (h = decompress(history))) {
    console.log("聊天记录压缩比: " + (history.length / h.length).toFixed(2));
    try {
      context.value = JSON.parse(h);
    } catch (error) {
      context.value = [];
      console.log(error);
    }
  }
  if (!maxOutputTokens.value) {
    localStorage.setItem("maxOutputTokens", defaultMaxOutputTokens);
    maxOutputTokens.value = defaultMaxOutputTokens;
  }
  if (!temperature.value) {
    localStorage.setItem("temperature", defaultTemperature);
    temperature.value = defaultTemperature;
  }

  nextTick(() => rollToBottom(false));

  if (!apiKey.value) {
    alert.value = {
      show: true,
      text: "请设置API密钥!",
      onclick: () => {
        setApiKey();
      }
    };
    return;
  }

  initAI();
})

const initAI = () => {
  genAI = new GoogleGenerativeAI(apiKey.value);
  model = genAI.getGenerativeModel(
    {
      model: aiModel.value,
      systemInstruction: systemInstruction.value,
      tools: searchDisabled.value ? [] : [
        {
          googleSearch: {}
        }
      ],
      generationConfig: {
        maxOutputTokens: maxOutputTokens.value,
        temperature: temperature.value
      }
    },
    {
      baseUrl: baseUrl.value,
      apiVersion: "v1beta",
    }
  );
  chat = model.startChat({
    history: cloneArray(context.value)
  });
}

watch(submitDisabled, (newVal, oldVal) => {
  if (newVal) {
    loadingChatInterval = setInterval(() => {
      loadingChatIndex.value = (loadingChatIndex.value + 1) % 4;
    }, 300);
  } else {
    clearInterval(loadingChatInterval);
  }
})

watch(input, (newVal, oldVal) => {
  textarea.value.style.height = "1px";
  nextTick(() => {
    if (isScrollAtBottom()) {
      nextTick(() => rollToBottom());
    }
    textarea.value.style.height = textarea.value.scrollHeight + "px";
    toolbar.value.style.height = "calc(" + textarea.value.scrollHeight + "px + 6vw)";
    chatContainer.value.style.paddingBottom = "min(calc(40px + 6vw + " +
      (textarea.value.scrollHeight - 40).toFixed(0) + "px), calc(50vh + 6vw))";
    scrollbarContainer.value.style.height = "calc(100vh - 6vw - " + textarea.value.scrollHeight + "px)";
  });
})

watch(showMenu, (newVal, oldVal) => {
  if (newVal) {
    renderedContextLength.value = defaultRenderedContextStep;
    nextTick(() => rollToBottom());
  }
})

watch(showSetting, (newVal, oldVal) => {
  if (newVal) {
    renderedContextLength.value = defaultRenderedContextStep;
    nextTick(() => rollToBottom());
  }
})

watch(prompt, (newVal, oldVal) => {
  if (newVal.show) {
    renderedContextLength.value = defaultRenderedContextStep;
    nextTick(() => rollToBottom());
  }
})

watch(alert, (newVal, oldVal) => {
  if (newVal.show) {
    renderedContextLength.value = defaultRenderedContextStep;
    nextTick(() => rollToBottom());
  }
})

const handleScroll = () => {
  let element = chatContainer.value;
  throttledUpdateScrollBar();
  if (handleScrollLock) return;
  if (element.scrollTop < 10 && renderedContextLength.value < context.value.length) {
    handleScrollLock = true;
    renderedContextLength.value += defaultRenderedContextStep;
    const oldScrollHeight = element.scrollHeight;
    const oldScrollTop = element.scrollTop;
    element.style.overflow = "hidden";

    nextTick(() => {
      const newScrollHeight = element.scrollHeight;
      const scrollHeightDiff = newScrollHeight - oldScrollHeight;
      element.scrollTo({
        top: oldScrollTop + scrollHeightDiff,
        behavior: 'instant'
      });
      handleScrollLock = false;
      element.style.overflow = "auto";
    });
  }
}

const submit = async () => {
  if (!apiKey.value) {
    alert.value = {
      show: true,
      text: "请设置API密钥!",
      onclick: () => { }
    };
    return;
  }
  renderedContextLength.value = defaultRenderedContextStep;
  closeMenu();
  const t = input.value;
  submitDisabled.value = true;
  input.value = "";
  while (context.value.length > 0
    && context.value[context.value.length - 1].role == "user") {
    context.value.pop();
  }
  context.value.push({ id: context.value.length, role: "user", parts: [{ text: t }] });
  await nextTick(() => rollToBottom());

  try {
    const result = await chat.sendMessageStream(t);
    context.value.push({ id: context.value.length, role: "model", parts: [{ text: "" }] });
    await nextTick(() => rollToBottom());
    for await (const chunk of result.stream) {
      const chunkText = chunk.text();
      context.value[context.value.length - 1].parts[0].text += chunkText;
      if (isScrollAtBottom()) {
        await nextTick(() => rollToBottom());
      }
    }
    submitDisabled.value = false;
    localStorage.setItem("history", compress(JSON.stringify(context.value)));
  } catch (error) {
    submitDisabled.value = false;
    if (context.value[context.value.length - 1].role == "user" && input.value.length == 0) {
      input.value = t;
      context.value.pop();
    }
    alert.value = {
      show: true,
      text: "操作失败！\n\n" + error,
      onclick: () => { }
    };
  }
}

const isScrollAtBottom = () => {
  const element = chatContainer.value;
  const scrollTop = element.scrollTop;
  const threshold = 50;
  const bottomPosition = element.scrollHeight - element.clientHeight;
  return scrollTop >= bottomPosition - threshold;
}

const rollToBottom = () => {
  const element = chatContainer.value;
  element.style.overflow = "hidden";
  element.scrollTo({
    top: element.scrollHeight,
    behavior: 'instant'
  });
  element.style.overflow = "auto";
}

const cloneArray = (arr) => {
  let t = [];
  for (let i = 0; i < arr.length; i++) {
    t[i] = { role: arr[i].role, parts: arr[i].parts };
  }
  return t;
}

const clickMenu = () => {
  if (showMenu.value) {
    showMenu.value = false;
    return;
  }
  if (showSetting.value) {
    showSetting.value = false;
    return;
  }
  if (alert.value.show) {
    alert.value.show = false;
    return;
  }
  if (prompt.value.show) {
    prompt.value.show = false;
    return;
  }
  showMenu.value = true;
}

const closeMenu = () => {
  showMenu.value = false;
  showSetting.value = false;
  prompt.value.show = false;
  alert.value.show = false;
}

const updateScrollBar = () => {
  showScrollBar.value = true;
  const element = chatContainer.value;
  const parentElement = scrollBar.value.parentElement;
  const parentHeight = parentElement.offsetHeight;
  const scrollPercent = (element.scrollTop / (element.scrollHeight - element.clientHeight)) * 100;
  const scaledScrollPercent = scrollPercent * (parentHeight - 50) / parentHeight;
  if (scrollPercent > 0) {
    scrollBar.value.style.height = scaledScrollPercent + "%";
  } else {
    scrollBar.value.style.height = 0;
  }
  clearTimeout(scrollbarHideTimer);
  scrollbarHideTimer = setTimeout(() => {
    showScrollBar.value = false;
  }, 700);
}

const throttledUpdateScrollBar = throttle(updateScrollBar, 3);

const clear = () => {
  input.value = "";
  context.value = [];
  chat = model.startChat({
    history: []
  });
  localStorage.removeItem("history");
}

const removeLast = () => {
  do {
    context.value.pop();
  } while (context.value.length > 0 &&
    context.value[context.value.length - 1].role == "user");

  chat = model.startChat({
    history: cloneArray(context.value)
  });
  localStorage.setItem("history", compress(JSON.stringify(context.value)));
}

const setBaseUrl = () => {
  prompt.value = {
    show: true,
    title: "设置API地址",
    text: baseUrl.value,
    onclick: () => {
      const url = prompt.value.text;
      if (!url) return;
      localStorage.setItem("baseUrl", url);
      baseUrl.value = url;
      initAI();
    }
  };
}

const setApiKey = () => {
  prompt.value = {
    show: true,
    title: "设置API密钥",
    text: apiKey.value,
    onclick: () => {
      const key = prompt.value.text;
      if (!key) return;
      localStorage.setItem("apiKey", key);
      apiKey.value = key;
      initAI();
    }
  };
}

const setModel = () => {
  prompt.value = {
    show: true,
    title: "设置模型",
    text: aiModel.value,
    onclick: () => {
      const m = prompt.value.text;
      if (!m) return;
      localStorage.setItem("model", m);
      aiModel.value = m;
      initAI();
    }
  };
}

const setSystemInstruction = () => {
  prompt.value = {
    show: true,
    title: "设置系统提示词",
    text: systemInstruction.value,
    onclick: () => {
      const si = prompt.value.text;
      if (!si) return;
      localStorage.setItem("systemInstruction", si);
      systemInstruction.value = si;
      initAI();
    }
  };
}

const setMaxOutputTokens = () => {
  prompt.value = {
    show: true,
    title: "设置输出令牌限制",
    text: maxOutputTokens.value,
    onclick: () => {
      const t = prompt.value.text;
      if (!t) return;
      localStorage.setItem("maxOutputTokens", t);
      maxOutputTokens.value = t;
      initAI();
    }
  };
}

const setTemperature = () => {
  prompt.value = {
    show: true,
    title: "设置温度",
    text: temperature.value,
    onclick: () => {
      const t = prompt.value.text;
      if (!t) return;
      localStorage.setItem("temperature", t);
      temperature.value = t;
      initAI();
    }
  };
}

const switchSearchDisabled = () => {
  searchDisabled.value = !searchDisabled.value;
  localStorage.setItem("searchDisabled", searchDisabled.value);
  initAI();
  alert.value = {
    show: true,
    text: "搜索已" + (searchDisabled.value ? "禁用" : "启用"),
    onclick: () => { }
  };
}

</script>

<style scoped>
.chat-container {
  width: 100%;
  padding: 3vw;
  border-radius: 5px;
  overflow-y: auto;
  overflow-x: hidden;
  height: 100vh;
  padding-bottom: calc(6vw + 40px);
}

.chat-container::-webkit-scrollbar {
  display: none;
  scrollbar-width: none;
  -ms-overflow-style: none;
}

.message-row {
  margin-bottom: 10px;
  display: flex;
  flex-direction: column;
  align-items: flex-start;
}

.user-message-row {
  align-items: flex-end;
  white-space: pre-line;
}

.message-bubble {
  display: inline-block;
  padding: 10px 15px;
  border-radius: 20px;
  margin-bottom: 5px;
  word-wrap: break-word;
  max-width: min(90%, 600px);
}

.user-bubble {
  background-color: #34c859;
  color: #fff;
  border-bottom-right-radius: 5px;
}

.assistant-bubble,
.menu-bubble {
  background-color: #ededed;
  color: #000;
  border-bottom-left-radius: 5px;
}

.menu-bubble {
  font-weight: bold;
  user-select: none;
  white-space: pre-line;
}

.menu-item {
  color: #2222ff;
  display: inline-block;
  margin-right: 10px;
  text-decoration: none;
  cursor: pointer;
}

.prompt-input {
  border: none;
  border-radius: 15px;
  padding: 5px;
}

.alert-text,
.prompt-title {
  margin-bottom: 10px;
}

.toolbar {
  position: fixed;
  width: 100vw;
  min-height: calc(40px + 6vw);
  bottom: 0;
  background: rgba(255, 255, 255, 0.85);
  backdrop-filter: blur(20px);
  will-change: backdrop-filter;
  max-height: calc(50vh + 6vw);
}

.input-box {
  width: calc(100% - 40px - 9vw);
  height: 40px;
  padding: 9px 38px 7px 10px;
  border: 1px solid #ddd;
  border-radius: 18px;
  position: absolute;
  bottom: 3vw;
  left: calc(40px + 6vw);
  max-height: 50vh;
  min-height: 40px;
  background: none;
}

.help-button {
  width: 40px;
  height: 40px;
  padding: 0px;
  background-color: #E9E9E9AA;
  color: #808080;
  border: none;
  border-radius: 50%;
  cursor: pointer;
  font-size: 16px;
  font-weight: bold;
  position: absolute;
  left: 3vw;
  bottom: 3vw;
  display: flex;
  justify-content: center;
  align-items: center;
}

.help-button:hover {
  background-color: #e3e2e2;
}

.submit-button {
  width: 30px;
  height: 30px;
  padding: 0px;
  background-color: #34c759;
  color: white;
  border: none;
  border-radius: 50%;
  cursor: pointer;
  font-size: 16px;
  font-weight: bold;
  position: absolute;
  right: calc(3vw + 5px);
  bottom: calc(3vw + 5px);
  display: flex;
  justify-content: center;
  align-items: center;
}

.submit-button:hover {
  background-color: #33bf56;
}

.scrollbar-container {
  position: fixed;
  right: 3px;
  top: 0;
  width: 3px;
  height: calc(100vh - 40px - 6vw);
  min-height: calc(50vh - 6vw);
}

.scrollbar-thumb {
  height: 0;
  max-height: calc(100% - 50px);
  transition: 50ms;
}

.scrollbar-track-bg {
  width: 100%;
  background-color: #999999;
  height: 50px;
  border-radius: 4px;
}
</style>

<style>
*,
*::before,
*::after {
  box-sizing: border-box;
}

html,
body {
  overflow: hidden;
  height: 100%;
  margin: 0px;
}

input:focus,
textarea:focus {
  outline: none;
}

input,
textarea {
  font-size: 16px;
  resize: none;
}

textarea::-webkit-scrollbar {
  width: 0px;
  height: 0px;
  background: transparent;
}

.v-leave-active {
  transition: opacity 0.4s ease;
}

.v-enter-from,
.v-leave-to {
  opacity: 0;
}
</style>
