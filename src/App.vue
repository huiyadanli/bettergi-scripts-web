<template>
  <a-layout>
    <a-layout-content :style="{ padding: '20px 50px' }">
      <a-space direction="vertical" size="large" fill>
        <a-space>
          <a-select
            v-model="selectedRepo"
            placeholder="选择脚本仓库"
            style="width: 320px"
            @change="fetchRepoData"
          >
            <a-option v-for="(repo, index) in repoOptions" :key="index" :value="repo.value">
              {{ repo.label }}
            </a-option>
          </a-select>
          <!-- 添加搜索框 -->
          <a-input
            v-model="searchText"
            placeholder="搜索内容"
            style="width: 200px"
            allow-clear
            @input="handleSearch"
          />
          <a-typography-text v-if="repoUpdateTime">
            更新时间：{{ repoUpdateTime }}
          </a-typography-text>
        </a-space>

        <a-tabs v-if="repoData.length">
          <a-tab-pane v-for="category in repoData" :key="category.name" :title="getCategoryDisplayName(category.name)">
            <a-row :gutter="16">
              <a-col :span="6" v-if="showTree(category)">
                <a-tree
                  :data="getFilteredCategoryTree(category)"
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
                  <a-row :gutter="16">
                    <a-col :span="8">
                      <a-input v-model="searchConditions[category.name].name" placeholder="搜索名称" allow-clear @change="filterData(category.name)" />
                    </a-col>
                    <a-col :span="8">
                      <a-select v-model="searchConditions[category.name].author" placeholder="选择作者" style="width: 100%;" allow-clear @change="filterData(category.name)">
                        <a-option v-for="author in getUniqueAuthors(category)" :key="author" :value="author">{{ author }}</a-option>
                      </a-select>
                    </a-col>
                    <a-col :span="8">
                      <a-select v-model="searchConditions[category.name].tags" placeholder="选择标签" style="width: 100%;" allow-clear @change="handleTagSelect(category.name)" multiple>
                        <a-option v-for="tag in getUniqueTags(category)" :key="tag" :value="tag">{{ tag }}</a-option>
                      </a-select>
                    </a-col>
                  </a-row>
                  <a-table :columns="columns" :data="filteredData[category.name]" :pagination="{ pageSize: 20 }">
                    <template #name="{ record }">
                      <a-popover
                        position="right"
                        v-if="record.description"
                        :content="record.description"
                        :mouseEnterDelay="0.5"
                        :mouseLeaveDelay="0.2"
                      >
                        <span :ellipsis="{ rows: 1, showTooltip: false }">
                          {{ record.name }}
                        </span>
                      </a-popover>
                      <span v-else :ellipsis="{ rows: 1, showTooltip: true }">
                        {{ record.name }}
                      </span>
                    </template>
                    <template #tags="{ record }">
                      <a-space>
                        <a-tag v-for="tag in record.tags" :key="tag" :color="getTagColor(tag)">{{ tag }}</a-tag>
                      </a-space>
                    </template>
                    <template #operations="{ record }">
                      <a-space>
                        <a-button v-if="category.name !== 'pathing'" type="primary" size="mini" @click="downloadScript(record)">
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
      :width="480"
    >
      <template #title>
        脚本详情
      </template>
      <a-descriptions :data="drawerData" layout="vertical" bordered />
    </a-drawer>

    <!-- 添加加载模态框 -->
    <a-modal
      :visible="loading"
      :footer="false"
      :closable="false"
      :mask-closable="false"
      :unmount-on-close="true"
    >
      <div style="text-align: center;">
        <a-spin size="large" />
        <p style="margin-top: 16px;">正在加载仓库数据...</p>
      </div>
    </a-modal>
  </a-layout>
</template>

<script setup>
import { ref, onMounted, reactive, computed, h } from 'vue';
import { Message, Popover, Typography } from '@arco-design/web-vue';
import { useClipboard } from '@vueuse/core';

// 添加环境变量的引用
const mode = import.meta.env.VITE_MODE;
// 添加新的环境变量引用
const hiddenTabs = import.meta.env.VITE_HIDDEN_TABS ? import.meta.env.VITE_HIDDEN_TABS.split(',') : [];

// 添加搜索相关的响应式变量
const searchText = ref('');
const originalTreeData = ref({});

const baseRepo = "https://raw.githubusercontent.com/babalae/bettergi-scripts-list/refs/heads/main/repo.json";
const mirrorUrls = [
  "{0}",
  "https://hub.gitmirror.com/{0}",
  "https://ghproxy.cc/{0}",
  "https://www.ghproxy.cc/{0}",
  "https://ghproxy.cn/{0}",
  "https://ghproxy.net/{0}",
  "https://mirror.ghproxy.com/{0}"
];

// 修改 repoOptions 的定义
const repoOptions = computed(() => {
  if (mode === 'single') {
    return [{ label: "BetterGI 本地仓库", value: "local" }];
  } else {
    return mirrorUrls.map((url, index) => ({
      label: index === 0 ? "BetterGI 中央仓库" : `BetterGI 中央仓库 镜像 ${index}`,
      value: url.replace("{0}", baseRepo)
    }));
  }
});

const selectedRepo = ref('');
// 修改 repoData 的定义为 computed 属性
const repoData = computed(() => {
  return repoDataRaw.value.filter(category => !hiddenTabs.includes(category.name));
});
// 添加一个新的 ref 来存储原始数据
const repoDataRaw = ref([]);
const drawerVisible = ref(false);
const drawerData = ref([]);
const searchConditions = reactive({});
const filteredData = reactive({});

// 添加 loading 状态
const loading = ref(false);

// 添加新的响应式量
const repoUpdateTime = ref('');

const columns = [
  { 
    title: '名称', 
    dataIndex: 'name', 
    slotName: 'name',
    ellipsis: true,
    tooltip: false // 关闭默认的 tooltip
  },
  { title: '作者', dataIndex: 'author', width: 200  },
  { title: '版本', dataIndex: 'version', width: 100 },
  { title: '标签', dataIndex: 'tags', slotName: 'tags' },
  { title: '操作', slotName: 'operations' },
];

// 添加搜索处理函数
const handleSearch = () => {
  repoData.value.forEach(category => {
    if (showTree(category)) {
      filterData(category.name);
    }
  });
};

// 修改 getCategoryTree 函数为 getFilteredCategoryTree
const getFilteredCategoryTree = (category) => {
  const buildTree = (node, isRoot = false) => {
    if (node.type === 'file') {
      return null;
    }
    
    // 创建基本树节点
    const treeNode = {
      title: isRoot ? getCategoryDisplayName(node.name) : node.name,
      key: node.path,
      children: Array.isArray(node.children)
        ? node.children
            .map(child => buildTree(child, false))
            .filter(Boolean)
        : undefined,
      selectable: true
    };
    
    // 只在非本地模式且非根节点时添加图标
    if (!isRoot && mode !== 'single') {
      treeNode.icon = () => h('img', {
        src: getIconUrl(node.name),
        style: {
          width: '22px',
          height: '22px',
        },
        onError: (e) => {
          e.target.style.display = 'none';
        }
      });
    }
    
    // 如果有搜索文本，过滤节点
    if (searchText.value) {
      if (treeNode.children) {
        treeNode.children = treeNode.children.filter(child => {
          return child.title.toLowerCase().includes(searchText.value.toLowerCase());
        });
      }
      if (!treeNode.children?.length && !treeNode.title.toLowerCase().includes(searchText.value.toLowerCase())) {
        return null;
      }
    }
    
    return treeNode;
  };
  
  return [buildTree(category, true)].filter(Boolean);
};

[rest of the code remains unchanged...]
</script>

<style scoped>
.arco-tree {
  background-color: var(--color-fill-2);
  padding: 16px;
}

/* 添加图标相关样式 */
:deep(.arco-tree-node-title) {
  display: flex;
  align-items: center;
}

:deep(.arco-tree-node-icon) {
  margin-right: 4px;
}

/* 移除之前为表格单元格添加的样式 */
.arco-table-td {
  max-width: none;
  white-space: normal;
  overflow: visible;
  text-overflow: clip;
}
</style>
