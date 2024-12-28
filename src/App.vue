<template>
  <a-layout :class="isDarkMode ? 'dark-mode' : 'light-mode'">
    <a-layout-header class="header">
      <a-button @click="toggleDarkMode">
        {{ isDarkMode ? '切换白天模式' : '切换黑夜模式' }}
      </a-button>
    </a-layout-header>
    <a-layout-content :style="{ padding: '20px 50px' }">
      <a-space direction="vertical" size="large" fill>
        <a-space>
          <a-select
            v-model="selectedRepo"
            placeholder="选择脚本仓库"
            style="width: 320px"
            @change="fetchRepoData"
            :class="isDarkMode ? 'dark-input' : 'light-input'"
          >
            <a-option v-for="(repo, index) in repoOptions" :key="index" :value="repo.value">
              {{ repo.label }}
            </a-option>
          </a-select>
          <!-- 添加搜索框 -->
          <a-input
            v-model="treeSearchText"
            placeholder="搜索左侧栏"
            style="width: 200px"
            allow-clear
            @input="handleTreeSearch"
            :class="isDarkMode ? 'dark-input' : 'light-input'"
          />
          <a-typography-text v-if="repoUpdateTime" :class="isDarkMode ? 'dark-text' : 'light-text'">
            更新时间：{{ repoUpdateTime }}
          </a-typography-text>
        </a-space>

        <a-tabs v-if="repoData.length" :class="isDarkMode ? 'dark-tabs' : 'light-tabs'">
          <a-tab-pane v-for="category in repoData" :key="category.name" :title="getCategoryDisplayName(category.name)">
            <a-row :gutter="16">
              <a-col :span="6" v-if="showTree(category)">
                <a-tree
                  :data="filteredTreeData[category.name] || getCategoryTree(category)"
                  :defaultExpandedKeys="getExpandedKeys(category)"
                  @select="(selectedKeys, event) => handleTreeSelect(selectedKeys, event, category.name)"
                  :class="isDarkMode ? 'dark-tree' : 'light-tree'"
                >
                  <template #extra="nodeData">
                    <a-button
                      type="text"
                      size="mini"
                      style="position: absolute; right: 8px; top: 6px;"
                      :class="isDarkMode ? 'dark-button' : 'light-button'"
                      @click.stop="() => onTreeIconClick(nodeData)"
                    >
                      订阅
                    </a-button>
                  </template>
                </a-tree>
              </a-col>
              <a-col :span="showTree(category) ? 18 : 24">
                <a-space direction="vertical" size="medium" style="width: 100%;">
                  <a-row :gutter="16">
                    <a-col :span="8">
                      <a-input 
                        v-model="searchConditions[category.name].name" 
                        placeholder="搜索名称" 
                        allow-clear 
                        @change="filterData(category.name)"
                        :class="isDarkMode ? 'dark-input' : 'light-input'"
                      />
                    </a-col>
                    <a-col :span="8">
                      <a-select 
                        v-model="searchConditions[category.name].author" 
                        placeholder="选择作者" 
                        style="width: 100%;" 
                        allow-clear 
                        @change="filterData(category.name)"
                        :class="isDarkMode ? 'dark-input' : 'light-input'"
                      >
                        <a-option v-for="author in getUniqueAuthors(category)" :key="author" :value="author">{{ author }}</a-option>
                      </a-select>
                    </a-col>
                    <a-col :span="8">
                      <a-select 
                        v-model="searchConditions[category.name].tags" 
                        placeholder="选择标签" 
                        style="width: 100%;" 
                        allow-clear 
                        @change="handleTagSelect(category.name)" 
                        multiple
                        :class="isDarkMode ? 'dark-input' : 'light-input'"
                      >
                        <a-option v-for="tag in getUniqueTags(category)" :key="tag" :value="tag">{{ tag }}</a-option>
                      </a-select>
                    </a-col>
                  </a-row>
                  <a-table 
                    :columns="columns" 
                    :data="filteredData[category.name]" 
                    :pagination="{ pageSize: 20 }"
                    :class="isDarkMode ? 'dark-table' : 'light-table'"
                  >
                    <!-- Table templates remain the same -->
                  </a-table>
                </a-space>
              </a-col>
            </a-row>
          </a-tab-pane>
        </a-tabs>

        <a-empty v-else description="请选择一个仓库" :class="isDarkMode ? 'dark-text' : 'light-text'" />
      </a-space>
    </a-layout-content>

    <!-- Drawer and Modal components remain the same -->
  </a-layout>
</template>

<script setup>
// Previous imports remain the same
import { ref, onMounted, reactive, computed, h } from 'vue';
import { Message, Popover, Typography } from '@arco-design/web-vue';
import { useClipboard } from '@vueuse/core';

// Add dark mode state
const isDarkMode = ref(false);

// Add dark mode toggle function
const toggleDarkMode = () => {
  isDarkMode.value = !isDarkMode.value;
};

// Rest of the script remains the same
</script>

<style scoped>
/* Dark mode styles */
.dark-mode {
  background-color: #1a1a1a;
  color: #ffffff;
}

.light-mode {
  background-color: #ffffff;
  color: #000000;
}

.header {
  padding: 0 20px;
  display: flex;
  justify-content: flex-end;
  align-items: center;
  height: 60px;
}

.dark-input {
  background-color: #2a2a2a !important;
  color: #ffffff !important;
  border-color: #3a3a3a !important;
}

.light-input {
  background-color: #ffffff !important;
  color: #000000 !important;
  border-color: #d9d9d9 !important;
}

.dark-text {
  color: #ffffff !important;
}

.light-text {
  color: #000000 !important;
}

.dark-tree {
  background-color: #2a2a2a !important;
  color: #ffffff !important;
}

.light-tree {
  background-color: var(--color-fill-2) !important;
}

.dark-table {
  background-color: #2a2a2a !important;
  color: #ffffff !important;
}

.dark-table :deep(th),
.dark-table :deep(td) {
  background-color: #2a2a2a !important;
  color: #ffffff !important;
  border-color: #3a3a3a !important;
}

.light-table :deep(th),
.light-table :deep(td) {
  background-color: #ffffff !important;
  color: #000000 !important;
}

.dark-tabs :deep(.arco-tabs-header) {
  border-bottom-color: #3a3a3a !important;
}

.dark-tabs :deep(.arco-tabs-tab) {
  color: #ffffff !important;
}

.dark-button {
  color: #3370ff !important;
}

.light-button {
  color: #3370ff !important;
}

/* Keep existing styles */
.arco-tree {
  padding: 16px;
}

:deep(.arco-tree-node-title) {
  display: flex;
  align-items: center;
}

:deep(.arco-tree-node-icon) {
  margin-right: 4px;
}

.arco-table-td {
  max-width: none;
  white-space: normal;
  overflow: visible;
  text-overflow: clip;
}
</style>
