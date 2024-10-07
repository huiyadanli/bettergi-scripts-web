<template>
  <a-layout>
    <a-layout-content :style="{ padding: '20px 50px' }">
      <a-space direction="vertical" size="large" fill>
        <a-select
          v-model="selectedRepo"
          placeholder="选择脚本仓库"
          style="width: 320px"
          @change="fetchRepoData"
        >
          <a-option value="https://raw.githubusercontent.com/babalae/bettergi-scripts-list/refs/heads/main/repo/items.json">BetterGI 中央仓库</a-option>
        </a-select>

        <a-tabs v-if="repoData.length">
          <a-tab-pane v-for="category in repoData" :key="category.type" :title="getTabTitle(category.type)">
            <a-table :columns="columns" :data="category.list" :pagination="{ pageSize: 20 }">
              <template #name="{ record }">
                  {{ record.name }}
              </template>
              <template #tags="{ record }">
                <a-space>
                  <a-tag v-for="tag in record.tags" :key="tag" color="blue">{{ tag }}</a-tag>
                </a-space>
              </template>
              <template #operations="{ record }">
                <a-space>
                  <a-button type="primary" size="mini" @click="downloadScript(record)">
                    下载
                  </a-button>
                  <a-button size="mini" @click="showDetails(record)">
                    详情
                  </a-button>
                </a-space>
              </template>
            </a-table>
          </a-tab-pane>
        </a-tabs>

        <a-empty v-else description="请选择一个仓库" />
      </a-space>
    </a-layout-content>

    <a-drawer
      :visible="drawerVisible"
      @cancel="closeDrawer"
      @ok="closeDrawer"
      unmountOnClose
    >
      <template #title>
        脚本详情
      </template>
      <a-descriptions :data="drawerData" layout="vertical" bordered />
    </a-drawer>
  </a-layout>
</template>

<script setup>
import { ref, onMounted } from 'vue';
import { Message } from '@arco-design/web-vue';

const selectedRepo = ref('');
const repoData = ref([]);
const drawerVisible = ref(false);
const drawerData = ref([]);

const columns = [
  { title: '名称', dataIndex: 'name', slotName: 'name' },
  { title: '作者', dataIndex: 'author' },
  { title: '版本', dataIndex: 'version' },
  { title: '标签', dataIndex: 'tags', slotName: 'tags' },
  { title: '操作', slotName: 'operations' },
];

const getTabTitle = (type) => {
  const titles = {
    js: 'JS脚本',
    pathing: '地图追踪',
    macro: '键鼠脚本',
    combat: '战斗策略',
    tcg: '七圣召唤策略',
    onekey: '一键宏',
  };
  return titles[type] || type;
};

const fetchRepoData = async () => {
  if (!selectedRepo.value) return;
  
  try {
    const response = await fetch(selectedRepo.value);
    const data = await response.json();
    repoData.value = data;
  } catch (error) {
    Message.error('获取仓库数据失败');
    console.error('Error fetching repo data:', error);
  }
};

onMounted(() => {
  // 默认选中第一个仓库
  selectedRepo.value = 'https://raw.githubusercontent.com/babalae/bettergi-scripts-list/refs/heads/main/repo/items.json';
  fetchRepoData();
});

const downloadScript = (script) => {
  // 实现下载逻辑
  console.log('Downloading script:', script.name);
  Message.success(`正在下载 ${script.name}`);
};

const showDetails = (script) => {
  drawerData.value = [
    { label: '名称', value: script.name },
    { label: '作者', value: script.author },
    { label: '版本', value: script.version },
    { label: '描述', value: script.description || '无描述' },
    { label: '路径', value: script.path },
    { label: '标签', value: script.tags },
    { label: 'Hash', value: script.hash },
  ];
  drawerVisible.value = true;
};

const closeDrawer = () => {
  drawerVisible.value = false;
};
</script>

<style scoped>

</style>