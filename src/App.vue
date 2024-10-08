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
          <a-option v-for="(repo, index) in repoOptions" :key="index" :value="repo.value">
            {{ repo.label }}
          </a-option>
        </a-select>

        <a-tabs v-if="repoData.length">
          <a-tab-pane v-for="category in repoData" :key="category.name" :title="getCategoryDisplayName(category.name)">
            <a-row :gutter="16">
              <a-col :span="6" v-if="showTree(category)">
                <a-tree
                  :data="getCategoryTree(category)"
                  @select="(selectedKeys, event) => handleTreeSelect(selectedKeys, event, category.name)"
                >
                  <template #extra="nodeData">
                    <a-tooltip content="点击订阅目录">
                      <IconCopy
                        style="position: absolute; right: 8px; font-size: 12px; top: 10px; color: #3370ff; cursor: pointer;"
                        @click.stop="() => onTreeIconClick(nodeData)"
                      />
                    </a-tooltip>
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
import { ref, onMounted, reactive, computed } from 'vue';
import { Message, Popover, Typography } from '@arco-design/web-vue';
import { useClipboard } from '@vueuse/core';

const baseRepo = "https://raw.githubusercontent.com/babalae/bettergi-scripts-list/refs/heads/main/build/tree.json";
const mirrorUrls = [
  "{0}",
  "https://mirror.ghproxy.com/{0}",
  "https://hub.gitmirror.com/{0}",
  "https://ghproxy.cc/{0}",
  "https://www.ghproxy.cc/{0}",
  "https://ghproxy.cn/{0}",
  "https://ghproxy.net/{0}"
];

const repoOptions = ref(mirrorUrls.map((url, index) => ({
  label: index === 0 ? "BetterGI 中央仓库" : `BetterGI 中央仓库 镜像 ${index}`,
  value: url.replace("{0}", baseRepo)
})));

const selectedRepo = ref('');
const repoData = ref([]);
const drawerVisible = ref(false);
const drawerData = ref([]);
const searchConditions = reactive({});
const filteredData = reactive({});

// 添加 loading 状态
const loading = ref(false);

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

const fetchRepoData = async () => {
  if (!selectedRepo.value) return;
  
  loading.value = true; // 显示加载模态框
  
  // 清空现有数据
  repoData.value = [];
  Object.keys(searchConditions).forEach(key => {
    searchConditions[key] = {
      name: '',
      author: '',
      tags: [],
      path: ''
    };
  });
  Object.keys(filteredData).forEach(key => {
    filteredData[key] = [];
  });
  
  try {
    const response = await fetch(selectedRepo.value);
    const data = await response.json();
    
    // 为所有节点生成 path
    data.forEach(category => generatePaths(category));
    
    repoData.value = data;
    initializeSearchConditions();
    
    // 初始化 tagColorMap
    data.forEach(category => {
      traverseCategory(category, (item) => {
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
  } finally {
    loading.value = false; // 隐藏加载模态框
  }
};

// 新增函数：为所有节点生成 path
const generatePaths = (node, parentPath = '') => {
  const currentPath = parentPath ? `${parentPath}/${node.name}` : node.name;
  node.path = currentPath;
  
  if (node.type === 'directory' && Array.isArray(node.children)) {
    node.children.forEach(child => generatePaths(child, currentPath));
  }
};

const traverseCategory = (category, callback) => {
  if (category.type === 'file') {
    callback(category);
  } else if (category.type === 'directory' && Array.isArray(category.children)) {
    if (category.name === 'js') {
      category.children.forEach(child => {
        if (child.type === 'directory') {
          // 处理 JS 脚本
          if (child.description && child.description.includes('~|~')) {
            const [nameSuffix, newDescription] = child.description.split('~|~');
            child.name = `${child.name} - ${nameSuffix.trim()}`;
            child.description = newDescription.trim();
          }
          callback(child);
        } else {
          traverseCategory(child, callback);
        }
      });
    } else {
      category.children.forEach(child => traverseCategory(child, callback));
    }
  }
};

const getUniqueAuthors = (category) => {
  const authors = new Set();
  traverseCategory(category, (item) => {
    if (item.author) authors.add(item.author);
  });
  return [...authors];
};

const getUniqueTags = (category) => {
  const tags = new Set();
  traverseCategory(category, (item) => {
    if (Array.isArray(item.tags)) {
      item.tags.forEach(tag => tags.add(tag));
    }
  });
  return [...tags];
};

const filterData = (categoryName) => {
  const category = repoData.value.find(cat => cat.name === categoryName);
  const condition = searchConditions[categoryName];
  
  const filtered = [];
  traverseCategory(category, (item) => {
    const nameMatch = !condition.name || item.name.toLowerCase().includes(condition.name.toLowerCase());
    const authorMatch = !condition.author || item.author === condition.author;
    // 修改标签匹配逻辑
    const tagMatch = condition.tags.length === 0 || (Array.isArray(item.tags) && condition.tags.every(tag => item.tags.includes(tag)));
    const pathMatch = !condition.path || (item.path && item.path.startsWith(condition.path) && item.path !== condition.path);
    if (nameMatch && authorMatch && tagMatch && pathMatch && (item.type === 'file' || (category.name === 'js' && item.type === 'directory'))) {
      filtered.push(item);
    }
  });
  
  filteredData[categoryName] = filtered;
};

const initializeSearchConditions = () => {
  repoData.value.forEach(category => {
    searchConditions[category.name] = {
      name: '',
      author: '',
      tags: [],
      path: ''
    };
    filteredData[category.name] = [];
    traverseCategory(category, (item) => {
      filteredData[category.name].push(item);
    });
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

const { copy } = useClipboard();

const downloadScript = (script) => {
  // 创建一个包含脚本路径的数组
  const subscriptionData = [script.path];

  // 将数组转换为 JSON 字符串
  const jsonString = JSON.stringify(subscriptionData);

  // 将 JSON 字符串复制到剪贴板
  copy(jsonString).then(() => {
    Message.success(`已将 ${script.name} 的订阅信息复制到剪贴板`);
  }).catch((error) => {
    console.error('复制到剪贴板失败:', error);
    Message.error(`复制 ${script.name} 的订阅信息失败`);
  });
};

const showDetails = (script) => {
  drawerData.value = [
    { label: '名称', value: script.name },
    { label: '作者', value: script.author },
    { label: '版本', value: script.version },
    { label: '描述', value: script.description || '无描述' },
    { label: '标签', value: script.tags },
    { label: 'Hash', value: script.hash },
  ];
  drawerVisible.value = true;
};

const closeDrawer = () => {
  drawerVisible.value = false;
};

const getCategoryTree = (category) => {
  const buildTree = (node, isRoot = false) => {
    if (node.type === 'file') {
      return null;
    }
    return {
      title: isRoot ? getCategoryDisplayName(node.name) : node.name,
      key: node.path,
      children: Array.isArray(node.children)
        ? node.children
            .map(child => buildTree(child, false))
            .filter(Boolean)
        : undefined,
      selectable: true
    };
  };
  return [buildTree(category, true)].filter(Boolean);
};

const showTree = (category) => {
  return category.name === 'pathing';
};

const handleTreeSelect = (selectedKeys, event, categoryName) => {
  const selectedNode = event.node;
  searchConditions[categoryName].path = selectedNode.key;
  filterData(categoryName);
};

const handleTagSelect = (categoryName) => {
  filterData(categoryName);
};

// 添加类别名称映射
const categoryNameMap = {
  'pathing': '地图追踪',
  'js': 'JS脚本',
  'combat': '战斗策略',
  'tcg': '七圣召唤',
  'onekey': '一键宏'
};

// 添加获取显示名称的函数
const getCategoryDisplayName = (name) => {
  return categoryNameMap[name] || name;
};

const onTreeIconClick = (nodeData) => {
  downloadScript({name: nodeData.title, path: nodeData.key});
};

onMounted(() => {
  // 默认选中第一个仓库
  if (repoOptions.value.length > 0) {
    selectedRepo.value = repoOptions.value[0].value;
    fetchRepoData();
  }
});
</script>

<style scoped>
.arco-tree {
  background-color: var(--color-fill-2);
  padding: 16px;
}

/* 移除之前为表格单元格添加的样式 */
.arco-table-td {
  max-width: none;
  white-space: normal;
  overflow: visible;
  text-overflow: clip;
}
</style>