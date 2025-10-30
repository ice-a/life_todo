<template>
  <div class="app-container">
    <h1>人生清单</h1>
    <p class="subtitle">探索并完成你生命中的1000个第一次体验</p>
    
    <div class="search-container">
      <input
        v-model="searchTerm"
        type="text"
        class="search-input"
        placeholder="搜索你想做的事情..."
      />
    </div>
    
    <div class="filter-controls">
      <button 
        :class="{ active: currentFilter === 'all' }"
        @click="currentFilter = 'all'"
      >
        全部
      </button>
      <button 
        :class="{ active: currentFilter === 'completed' }"
        @click="currentFilter = 'completed'"
      >
        已完成
      </button>
      <button 
        :class="{ active: currentFilter === 'pending' }"
        @click="currentFilter = 'pending'"
      >
        未完成
      </button>
    </div>
    
    <div class="life-list-container">
      <div 
        v-for="item in paginatedItems" 
        :key="item.id"
        class="life-list-item"
        :class="{ completed: item.completed }"
      >
        <div class="item-content">
          <input 
            type="checkbox" 
            :checked="item.completed"
            @change="toggleItem(item.id)"
            class="checkbox"
          />
          <span>{{ item.text }}</span>
        </div>
        <div class="item-meta">
          <span class="category">{{ item.category }}</span>
        </div>
      </div>
    </div>
    
    <div v-if="filteredItems.length === 0" class="empty-state">
      没有找到匹配的人生清单项
    </div>
    
    <div v-if="filteredItems.length > 0" class="pagination">
      <button 
        @click="currentPage = 1"
        :disabled="currentPage === 1"
      >
        首页
      </button>
      <button 
        @click="currentPage--"
        :disabled="currentPage === 1"
      >
        上一页
      </button>
      <span>{{ currentPage }} / {{ totalPages }}</span>
      <button 
        @click="currentPage++"
        :disabled="currentPage === totalPages"
      >
        下一页
      </button>
      <button 
        @click="currentPage = totalPages"
        :disabled="currentPage === totalPages"
      >
        末页
      </button>
    </div>
    
    <div class="stats">
      <p>已完成: {{ completedCount }} / {{ totalItems }}</p>
      <p>完成率: {{ completionRate }}%</p>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'

export default {
  name: 'App',
  setup() {
    const lifeList = ref([])
    const searchTerm = ref('')
    const currentFilter = ref('all')
    const currentPage = ref(1)
    const itemsPerPage = 20

    // 计算属性
    const filteredItems = computed(() => {
      let filtered = lifeList.value
      
      // 搜索过滤
      if (searchTerm.value) {
        const term = searchTerm.value.toLowerCase()
        filtered = filtered.filter(item => 
          item.text.toLowerCase().includes(term) ||
          item.category.toLowerCase().includes(term)
        )
      }
      
      // 状态过滤
      if (currentFilter.value === 'completed') {
        filtered = filtered.filter(item => item.completed)
      } else if (currentFilter.value === 'pending') {
        filtered = filtered.filter(item => !item.completed)
      }
      
      return filtered
    })
    
    const totalItems = computed(() => lifeList.value.length)
    const completedCount = computed(() => 
      lifeList.value.filter(item => item.completed).length
    )
    const completionRate = computed(() => 
      totalItems.value > 0 ? Math.round((completedCount.value / totalItems.value) * 100) : 0
    )
    
    const totalPages = computed(() => 
      Math.ceil(filteredItems.value.length / itemsPerPage)
    )
    
    const paginatedItems = computed(() => {
      const start = (currentPage.value - 1) * itemsPerPage
      const end = start + itemsPerPage
      return filteredItems.value.slice(start, end)
    })
    
    // 方法
    const toggleItem = (id) => {
      const item = lifeList.value.find(item => item.id === id)
      if (item) {
        item.completed = !item.completed
        saveToLocalStorage()
      }
    }
    
    const saveToLocalStorage = () => {
      localStorage.setItem('lifeListCompleted', JSON.stringify(
        lifeList.value.map(item => ({ id: item.id, completed: item.completed }))
      ))
    }
    
    const loadFromLocalStorage = () => {
      const saved = localStorage.getItem('lifeListCompleted')
      if (saved) {
        const completedItems = JSON.parse(saved)
        completedItems.forEach(savedItem => {
          const item = lifeList.value.find(item => item.id === savedItem.id)
          if (item) {
            item.completed = savedItem.completed
          }
        })
      }
    }
    
    // 生命周期
    onMounted(async () => {
      try {
        // 尝试动态导入JSON数据
        const response = await fetch('/data/lifeList.json');
        const lifeListData = await response.json();
        
        // 初始化数据
        lifeList.value = lifeListData.map((item, index) => ({
          id: index + 1,
          text: item,
          category: item.includes('旅行') ? '旅行' : 
                    item.includes('学习') ? '学习' : 
                    item.includes('运动') ? '运动' : 
                    item.includes('美食') ? '美食' : '其他',
          completed: false
        }))
        
        // 加载已完成状态
        loadFromLocalStorage()
      } catch (error) {
        console.error('Failed to load life list data:', error);
        // 使用备用数据
        lifeList.value = [
          { id: 1, text: '环游世界', category: '旅行', completed: false },
          { id: 2, text: '攀登高山', category: '运动', completed: false },
          { id: 3, text: '学习一门新语言', category: '学习', completed: false }
        ];
      }
    })
    
    return {
      lifeList,
      searchTerm,
      currentFilter,
      currentPage,
      filteredItems,
      paginatedItems,
      totalPages,
      totalItems,
      completedCount,
      completionRate,
      toggleItem
    }
  }
}
</script>

<style>
.app-container {
  width: 100%;
  max-width: 900px;
  margin: 0 auto;
}

.subtitle {
  font-size: 1.2em;
  color: #666;
  margin-bottom: 30px;
}

.search-container {
  display: flex;
  justify-content: center;
  margin-bottom: 20px;
}

.item-content {
  display: flex;
  align-items: center;
  gap: 15px;
  flex: 1;
}

.item-meta {
  display: flex;
  gap: 10px;
}

.category {
  background: rgba(100, 108, 255, 0.2);
  padding: 4px 12px;
  border-radius: 12px;
  font-size: 0.8em;
  color: #646cff;
}

.empty-state {
  padding: 40px;
  text-align: center;
  color: #666;
}

.stats {
  margin-top: 30px;
  display: flex;
  justify-content: center;
  gap: 30px;
  font-size: 1.1em;
}

/* 确保按钮激活状态正确显示 */
.filter-controls button.active {
  background-color: #646cff;
  color: white;
}
</style>