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
            <a-row :gutter="16">
              <a-col :span="6" v-if="showTree(category)">
                <a-tree
                  :data="getCategoryTree(category)"
                  @select="(selectedKeys) => handleTreeSelect(selectedKeys, category.type)"
                  :selectedKeys="searchConditions[category.type].tags"
                  multiple
                />
              </a-col>
              <a-col :span="showTree(category) ? 18 : 24">
                <a-space direction="vertical" size="medium" style="width: 100%;">
                  <a-row :gutter="16">
                    <a-col :span="8">
                      <a-input v-model="searchConditions[category.type].name" placeholder="搜索名称" allow-clear @change="filterData(category.type)" />
                    </a-col>
                    <a-col :span="8">
                      <a-select v-model="searchConditions[category.type].author" placeholder="选择作者" style="width: 100%;" allow-clear @change="filterData(category.type)">
                        <a-option v-for="author in getUniqueAuthors(category.list)" :key="author" :value="author">{{ author }}</a-option>
                      </a-select>
                    </a-col>
                    <a-col :span="8">
                      <a-select v-model="searchConditions[category.type].tags" placeholder="选择标签" style="width: 100%;" allow-clear @change="handleTagSelect(category.type)" multiple>
                        <a-option v-for="tag in getUniqueTags(category.list)" :key="tag" :value="tag">{{ tag }}</a-option>
                      </a-select>
                    </a-col>
                  </a-row>
                  <a-table :columns="columns" :data="filteredData[category.type]" :pagination="{ pageSize: 20 }">
                    <template #name="{ record }">
                      {{ record.name }}
                    </template>
                    <template #tags="{ record }">
                      <a-space>
                        <a-tag v-for="tag in record.tags" :key="tag" :color="getTagColor(tag)">{{ tag }}</a-tag>
                      </a-space>
                    </template>
                    <template #operations="{ record }">
                      <a-space>
                        <a-button type="primary" size="mini" @click="downloadScript(record)">
                          订阅
                        </a-button>
                        <a-button size="mini" @click="showDetails(record)">
                          详情
                        </a-button>
                      </a-space>
                    </template>
                  </a-table>
                </a-space>
              </a-col>
            </a-row>
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
import { ref, onMounted, reactive, computed } from 'vue';
import { Message } from '@arco-design/web-vue';

const selectedRepo = ref('');
const repoData = ref([]);
const drawerVisible = ref(false);
const drawerData = ref([]);
const searchConditions = reactive({});
const filteredData = reactive({});

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
    initializeSearchConditions();
    
    // 初始化 tagColorMap
    data.forEach(category => {
      category.list.forEach(item => {
        if (Array.isArray(item.tags)) {
          item.tags.forEach(tag => {
            if (tag && typeof tag === 'string' && !tagColorMap[tag]) {
              tagColorMap[tag] = getRandomColor();
            }
          });
        }
      });
    });
  } catch (error) {
    Message.error('获取仓库数据失败');
    console.error('Error fetching repo data:', error);
  }
};

const getUniqueAuthors = (list) => {
  return [...new Set(list.map(item => item.author))];
};

const getUniqueTags = (list) => {
  return [...new Set(list.flatMap(item => item.tags))];
};

const filterData = (type) => {
  const condition = searchConditions[type];
  filteredData[type] = repoData.value.find(category => category.type === type).list.filter(item => {
    const nameMatch = !condition.name || item.name.toLowerCase().includes(condition.name.toLowerCase());
    const authorMatch = !condition.author || item.author === condition.author;
    const tagMatch = condition.tags.length === 0 || condition.tags.some(tag => item.tags.includes(tag));
    return nameMatch && authorMatch && tagMatch;
  });
};

const initializeSearchConditions = () => {
  repoData.value.forEach(category => {
    searchConditions[category.type] = {
      name: '',
      author: '',
      tags: []
    };
    filteredData[category.type] = category.list;
  });
};

const getRandomColor = () => {
  const letters = '0123456789ABCDEF';
  let color = '#';
  for (let i = 0; i < 6; i++) {
    color += letters[Math.floor(Math.random() * 16)];
  }
  return color;
};

const tagColorMap = reactive({});

const getTagColor = (tag) => {
  if (!tagColorMap[tag]) {
    tagColorMap[tag] = getRandomColor();
  }
  return tagColorMap[tag];
};

onMounted(() => {
  // 默认选中第一个仓库
  selectedRepo.value = 'https://raw.githubusercontent.com/babalae/bettergi-scripts-list/refs/heads/main/repo/items.json';
  fetchRepoData().then(() => {
    initializeSearchConditions();
  });
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

const getCategoryTree = (category) => {
  const tags = getUniqueTags(category.list);
  if (tags.length === 0) {
    return [];
  }
  if (category.type === 'tcg') {
    return [
    {
      title: getTabTitle(category.type),
      key: category.type,
      children: [{ title: '惊喜牌组', key: '惊喜牌组' }],
      selectable: false
    }
  ];
  } else if (category.type === 'pathing') {
    return [
    {
      title: getTabTitle(category.type),
      key: category.type,
      children: tags.map(tag => ({ title: tag, key: tag })),
      selectable: false
    }
  ];
  }
  return [];
};

const showTree = (category) => {
  if (getCategoryTree(category).length === 0) {
    return false;
  }

  return getUniqueTags(category.list).length > 1;
};

const handleTreeSelect = (selectedKeys, categoryType) => {
  searchConditions[categoryType].tags = selectedKeys.filter(key => key !== categoryType);
  filterData(categoryType);
};

const handleTagSelect = (categoryType) => {
  filterData(categoryType);
};
</script>

<style scoped>
.arco-tree {
  background-color: var(--color-fill-2);
  padding: 16px;
  /* border-radius: 4px; */
}
</style>