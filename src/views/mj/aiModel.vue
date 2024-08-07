<script setup lang="ts">
import { NSelect, NInput, NSlider, NButton, useMessage, NTag } from "naive-ui";
import { ref, computed, watch, onMounted } from "vue";
import { gptConfigStore, homeStore, useChatStore } from '@/store';
import { mlog, chatSetting } from "@/api";
import { t } from "@/locales";

const emit = defineEmits(['close']);
const chatStore = useChatStore();
const uuid = chatStore.active;
const chatSet = new chatSetting(uuid == null ? 1002 : uuid);

const nGptStore = ref(chatSet.getGptConfig());

import axios from 'axios';

const options = ref({});
const selectedValues = ref({});
const selectWidth = ref(0);

onMounted(() => {
  fetch(`/v1/models`)
    .then(response => {
      if (!response.ok) {
        throw new Error('请求失败');
      }
      return response.json();
    })
    .then(data => {
      if (Array.isArray(data.data)) {
        config.value.model = data.data.map(model => model.id);
      } else {
        console.error('返回的数据格式不正确');
      }
    })
    .catch(error => {
      console.error('获取模型列表失败：', error);
    });

  axios.get(`/api/prompts`)
    .then(response => {
      if (response.data.status === 'Success') {
        options.value = response.data.data;
      } else {
        console.error('获取数据失败：', response.data.message);
      }
    })
    .catch(error => {
      console.error('获取数据失败：', error);
    });

  calculateSelectWidth();
  window.addEventListener('resize', calculateSelectWidth);

  // 在组件初始化时手动触发一次watch
  if (nGptStore.value.model) {
    handleModelChange(nGptStore.value.model);
  }
});

const calculateSelectWidth = () => {
  const containerWidth = document.querySelector('.select-container')?.clientWidth || 0;
  selectWidth.value = containerWidth / 2 - 0; // Subtracting gap
};

const handleSelectChange = (folder: string) => {
  nGptStore.value.systemMessage = selectedValues.value[folder] || "";
  for (const key in selectedValues.value) {
    if (key !== folder) {
      selectedValues.value[key] = null;
    }
  }
};

const config = ref({
  model: [],
  maxToken: 64000
});
const st = ref({ openMore: false });
const voiceList = computed(() => {
  let rz = [];
  for (let o of "alloy,echo,fable,onyx,nova,shimmer".split(/[ ,]+/ig)) rz.push({ label: o, value: o })
  return rz;
});
const modellist = computed(() => {
  let rz = [];
  for (let o of config.value.model) {
    rz.push({ label: o, value: o })
  }
  if (gptConfigStore.myData.userModel) {
    let arr = gptConfigStore.myData.userModel.split(/[ ,]+/ig);
    for (let o of arr) {
      o && rz.push({ label: o, value: o })
    }
  }
  if (homeStore.myData.session.cmodels) {
    let delModel: string[] = [];
    let addModel: string[] = [];
    let isDelAll = false
    homeStore.myData.session.cmodels.split(/[ ,]+/ig).map((v: string) => {
      if (v.indexOf('-') == 0) {
        delModel.push(v.substring(1))
        if (v == '-all') isDelAll = true;
      } else {
        addModel.push(v);
      }
    });
    mlog('cmodels', delModel, addModel);
    if (isDelAll) rz = [];
    rz = rz.filter(v => delModel.indexOf(v.value) == -1);
    addModel.map(o => rz.push({ label: o, value: o }))
    if (rz.length == 0) {
      rz.push({ label: 'gpt-3.5-turbo', value: 'gpt-3.5-turbo' })
    }
  }

  let uniqueArray: { label: string, value: string }[] = Array.from(
    new Map(rz.map(item => [JSON.stringify(item), item]))
      .values()
  );
  return uniqueArray;
});
const ms = useMessage();
const saveChat = (type: string) => {
  chatSet.save(nGptStore.value);
  gptConfigStore.setMyData(nGptStore.value);
  homeStore.setMyData({ act: 'saveChat' });
  if (type != 'hide') ms.success(t('common.saveSuccess'));
  emit('close');
}

// 将watch的回调提取到一个函数中，方便复用
const handleModelChange = (n) => {
  console.log('Model changed to:', n); // 添加日志
  nGptStore.value.gpts = undefined; // 重置 gpts 数据
  let max = 64000; // 默认最大令牌数

  config.value.maxToken = max; // 设置最大令牌数为默认最大值
  nGptStore.value.max_tokens = Math.min(max, nGptStore.value.max_tokens); // 更新 max_tokens 为默认最大值，但不小于当前值
};

// 注册watch
watch(() => nGptStore.value.model, handleModelChange);

const reSet = () => {
  console.log('Resetting configuration'); // 添加日志
  gptConfigStore.setInit(); // 将 gptConfigStore 恢复到初始状态

  // 手动设置 nGptStore 的默认值，但不包括模型
  nGptStore.value = {
    ...nGptStore.value, // 保留模型的值
    max_tokens: 4096, // 添加 max_tokens 的默认值
    userModel: '',
    talkCount: 20,
    systemMessage: '',
    temperature: 0.5,
    top_p: 1,
    presence_penalty: 0,
    frequency_penalty: 0,
    tts_voice: "alloy"
  };

  // 手动触发一次watch，确保watch能够监听到model的变化
  if (nGptStore.value.model) {
    handleModelChange(nGptStore.value.model);
  }
};

const filterModelInput = ref('');
const filteredModelList = computed(() => {
  return modellist.value.filter(model => model.label.includes(filterModelInput.value));
});
</script>

<template>
  <section class="mb-2 flex justify-between items-center">
    <div class="flex items-center w-1/2">
      <span class="text-red-500">*</span> {{ $t('mjset.model') }}
      <n-input v-model:value="filterModelInput" placeholder="筛选模型" size="small" class="ml-2" style="height: 100%;" />
    </div>
    <n-select v-model:value="nGptStore.model" :options="filteredModelList" size="small" class="w-1/2 ml-2" style="height: 100%;" />
  </section>
  <section class="mb-0 flex justify-between items-center">
    <n-input :placeholder="$t('mjchat.modlePlaceholder')" v-model:value="gptConfigStore.myData.userModel">
      <template #prefix>
        {{ $t('mjchat.myModle') }}
      </template>
    </n-input>
  </section>
  <section class="flex justify-between items-center">
    <div>{{ $t('mjchat.historyCnt') }}</div>
    <div class="flex justify-end items-center w-[80%] max-w-[240px]">
      <div class="w-[200px]"><n-slider v-model:value="nGptStore.talkCount" :step="1" :max="50" /></div>
      <div class="w-[40px] text-right">{{ nGptStore.talkCount }}</div>
    </div>
  </section>
  <div class="mb-0 text-[12px] text-gray-300 dark:text-gray-300/20">{{ $t('mjchat.historyToken') }}</div>

  <section class="flex justify-between items-center">
    <div>{{ $t('mjchat.historyTCnt') }}</div>
    <div class="flex justify-end items-center w-[80%] max-w-[240px]">
      <div class="w-[200px]"><n-slider v-model:value="nGptStore.max_tokens" :step="1" :max="config.maxToken" :min="1" /></div>
      <div class="w-[40px] text-right">{{ nGptStore.max_tokens }}</div>
    </div>
  </section>
  <div class="mb-0 text-[12px] text-gray-300 dark:text-gray-300/20">{{ $t('mjchat.historyTCntInfo') }}</div>

  <section class="mb-0">
    <div>{{ $t('mjchat.role') }}</div>
    <div>
      <n-input type="textarea" :placeholder="$t('mjchat.rolePlaceholder')" v-model:value="nGptStore.systemMessage" :autosize="{ minRows: 1, maxRows: 3 }" style="overflow-y: auto;" />
    </div>
  </section>

  <div class="select-container">
    <div v-for="(items, folder) in options" :key="folder" class="mb-0">
      <h3>{{ folder }}</h3>
      <n-select v-model:value="selectedValues[folder]" :options="items.map(item => ({ label: item.title, value: item.systemRole }))" @update:value="handleSelectChange(folder)" :style="{ maxWidth: selectWidth + 'px' }" />
    </div>
  </div>

  <template v-if="st.openMore">
    <section class="flex justify-between items-center">
      <div>{{ $t('mj.temperature') }}</div>
      <div class="flex justify-end items-center w-[80%] max-w-[240px]">
        <div class="w-[200px]"><n-slider v-model:value="nGptStore.temperature" :step="0.01" :max="1" /></div>
        <div class="w-[40px] text-right">{{ nGptStore.temperature }}</div>
      </div>
    </section>
    <div class="mb-0 text-[12px] text-gray-300 dark:text-gray-300/20">{{ $t('mj.temperatureInfo') }}</div>

    <section class="flex justify-between items-center">
      <div>{{ $t('mj.top_p') }}</div>
      <div class="flex justify-end items-center w-[80%] max-w-[240px]">
        <div class="w-[200px]"><n-slider v-model:value="nGptStore.top_p" :step="0.01" :max="1" /></div>
        <div class="w-[40px] text-right">{{ nGptStore.top_p }}</div>
      </div>
    </section>
    <div class="mb-0 text-[12px] text-gray-300 dark:text-gray-300/20">{{ $t('mj.top_pInfo') }}</div>

    <section class="flex justify-between items-center">
      <div>{{ $t('mj.presence_penalty') }}</div>
      <div class="flex justify-end items-center w-[80%] max-w-[240px]">
        <div class="w-[200px]"><n-slider v-model:value="nGptStore.presence_penalty" :step="0.01" :max="1" /></div>
        <div class="w-[40px] text-right">{{ nGptStore.presence_penalty }}</div>
      </div>
    </section>
    <div class="mb-0 text-[12px] text-gray-300 dark:text-gray-300/20">{{ $t('mj.presence_penaltyInfo') }}</div>

    <section class="flex justify-between items-center">
      <div>{{ $t('mj.frequency_penalty') }}</div>
      <div class="flex justify-end items-center w-[80%] max-w-[240px]">
        <div class="w-[200px]"><n-slider v-model:value="nGptStore.frequency_penalty" :step="0.01" :max="1" /></div>
        <div class="w-[40px] text-right">{{ nGptStore.frequency_penalty }}</div>
      </div>
    </section>
    <div class="mb-0 text-[12px] text-gray-300 dark:text-gray-300/20">{{ $t('mj.frequency_penaltyInfo') }}</div>

    <section class="mb-2 flex justify-between items-center">
      <div>{{ $t('mj.tts_voice') }}</div>
      <n-select v-model:value="nGptStore.tts_voice" :options="voiceList" size="small" class="!w-[50%]" />
    </section>
  </template>
  <div v-else class="text-right cursor-pointer mb-2" @click="st.openMore = true">
    <NTag type="primary" round size="small" :bordered="false" class="!cursor-pointer">More...</NTag>
  </div>

  <section class="text-right flex justify-end space-x-2">
    <NButton @click="reSet()">{{ $t('mj.setBtBack') }}</NButton>
    <NButton type="primary" @click="saveChat('no')">{{ $t('common.save') }}</NButton>
  </section>
</template>

<style scoped>
.select-container {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 5px;
}

.n-input, .n-select {
  height: 100%;
}
</style>
