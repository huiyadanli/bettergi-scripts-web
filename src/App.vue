<template>
  <a-layout>
    <a-layout-content :style="{ padding: '20px 50px' }">
      <a-space direction="vertical" size="large" fill>
        <!-- 新增全局搜索框 -->
        <a-input
          v-model="globalSearchQuery"
          placeholder="全局搜索"
          allow-clear
          @input="handleGlobalSearch"
          style="width: 100%; margin-bottom: 16px;"
        />

        <a-space>
          <a-select
            v-model="selectedRepo"
            placeholder="选择脚本仓库"
            style="width: 320px"
            @change="fetchRepoData"
          >
            <a-option
              v-for="(repo, index) in repoOptions"
              :key="index"
              :value="repo.value"
            >
              {{ repo.label }}
            </a-option>
          </a-select>
          <a-typography-text v-if="repoUpdateTime">
            更新时间：{{ repoUpdateTime }}
          </a-typography-text>
        </a-space>

        <!-- 原有的 a-tabs 和内容 -->
        <a-tabs v-if="filteredRepoData.length">
          <a-tab-pane
            v-for="category in filteredRepoData"
            :key="category.name"
            :title="getCategoryDisplayName(category.name)"
          >
            <a-row :gutter="16">
              <a-col :span="6" v-if="showTree(category)">
                <a-tree
                  :data="getCategoryTree(category)"
                  :defaultExpandedKeys="getExpandedKeys(category)"
                  @select="(selectedKeys, event) => handleTreeSelect(selectedKeys, event, category.name)"
                >
                  <template #extra="nodeData">
                    <a-button
                      type="text"
                      size="mini"
                      style="position: absolute; right: 8px; top: 6px; color: #3370ff;"
                      @click.stop="() => onTreeIconClick(nodeData)"
                    >
                      订阅
                    </a-button>
                  </template>
                </a-tree>
              </a-col>
              <a-col :span="showTree(category) ? 18 : 24">
                <a-space direction="vertical" size="medium" style="width: 100%;">
                  <!-- 内部搜索框 -->
                  <a-input
                    v-model="category.searchQuery"
                    placeholder="搜索当前类别"
                    allow-clear
                    style="width: 100%; margin-bottom: 8px;"
                  />
                  <!-- 表格显示搜索结果 -->
                  <a-table
                    :columns="categoryColumns"
                    :data-source="getFilteredCategoryItems(category)"
                    :pagination="{ pageSize: 10 }"
                  ></a-table>
                </a-space>
              </a-col>
            </a-row>
          </a-tab-pane>
        </a-tabs>

        <a-empty v-else description="请选择一个仓库" />
      </a-space>
    </a-layout-content>
  </a-layout>
</template>

<script setup>
import { ref, reactive, computed } from 'vue';

// 数据源和全局搜索
const globalSearchQuery = ref(''); // 全局搜索框绑定的变量
const selectedRepo = ref(null);
const repoData = reactive([]); // 仓库数据
const repoOptions = ref([
  { label: '仓库 1', value: 'repo1' },
  { label: '仓库 2', value: 'repo2' }
]);
const repoUpdateTime = ref('');

// 根据搜索查询过滤仓库数据
const filteredRepoData = computed(() => {
  if (!globalSearchQuery.value.trim()) {
    return repoData; // 如果没有输入搜索词，返回原始数据
  }

  const query = globalSearchQuery.value.toLowerCase();
  return repoData.filter(category => {
    let matches = false;
    traverseCategory(category, item => {
      if (
        (item.name && item.name.toLowerCase().includes(query)) ||
        (item.description && item.description.toLowerCase().includes(query)) ||
        (item.author && item.author.toLowerCase().includes(query)) ||
        (item.tags && item.tags.some(tag => tag.toLowerCase().includes(query)))
      ) {
        matches = true;
      }
    });
    return matches;
  });
});

// 获取分类中匹配的项
const getFilteredCategoryItems = category => {
  if (!category.searchQuery) return category.items;
  const query = category.searchQuery.toLowerCase();
  return category.items.filter(item =>
    item.name.toLowerCase().includes(query) ||
    (item.description && item.description.toLowerCase().includes(query))
  );
};

// 工具方法：遍历分类
const traverseCategory = (category, callback) => {
  if (category.items) {
    category.items.forEach(item => callback(item));
  }
  if (category.children) {
    category.children.forEach(child => traverseCategory(child, callback));
  }
};

// 模拟获取仓库数据
const fetchRepoData = value => {
  // 模拟根据仓库 ID 获取数据
  repoData.splice(0, repoData.length, ...mockFetchRepoData(value));
  repoUpdateTime.value = new Date().toLocaleString();
};

const mockFetchRepoData = repoId => [
  {
    name: '分类 A',
    items: [
      { name: '脚本 1', description: '这是脚本 1 的描述', author: '作者 A', tags: ['tag1'] },
      { name: '脚本 2', description: '这是脚本 2 的描述', author: '作者 B', tags: ['tag2'] }
    ]
  },
  {
    name: '分类 B',
    items: [
      { name: '脚本 3', description: '这是脚本 3 的描述', author: '作者 C', tags: ['tag3'] }
    ]
  }
];
</script>

<style scoped>
/* 样式可根据需要调整 */
</style>
