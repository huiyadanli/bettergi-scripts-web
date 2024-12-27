<template>
  <div id="app">
    <!-- 全局搜索框 -->
    <div class="search-container">
      <input
        type="text"
        v-model="searchQuery"
        placeholder="全局搜索..."
        @input="performSearch"
      />
    </div>

    <!-- 搜索结果展示 -->
    <div v-if="searchResults.length > 0" class="search-results">
      <h3>搜索结果：</h3>
      <ul>
        <li v-for="(result, index) in searchResults" :key="index">
          {{ result }}
        </li>
      </ul>
    </div>

    <!-- 现有内容 -->
    <router-view />
  </div>
</template>

<script>
export default {
  data() {
    return {
      searchQuery: "",
      searchResults: [],
      dataSources: {
        地图追踪: ["万相石", "压丘王", "刀耀"],
        JS脚本: ["脚本A", "脚本B"],
        战斗策略: ["策略1", "策略2"],
        七圣召唤: ["召唤术1", "召唤术2"]
      }
    };
  },
  methods: {
    performSearch() {
      const query = this.searchQuery.toLowerCase();
      const results = [];

      // 遍历数据源进行搜索
      for (const [category, items] of Object.entries(this.dataSources)) {
        items.forEach(item => {
          if (item.toLowerCase().includes(query)) {
            results.push(`${category}: ${item}`);
          }
        });
      }

      this.searchResults = results;
    }
  }
};
</script>

<style>
.search-container {
  margin: 16px;
  text-align: center;
}

.search-container input {
  width: 60%;
  padding: 8px;
  font-size: 16px;
}

.search-results {
  margin: 16px;
}

.search-results ul {
  list-style-type: none;
  padding: 0;
}

.search-results li {
  margin: 4px 0;
  font-size: 14px;
}
</style>
