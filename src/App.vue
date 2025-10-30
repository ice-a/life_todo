<template>
  <div class="app-container">
    <h1>人生清单</h1>
    <p class="subtitle">探索并完成你生命中的1000个第一次体验</p>
    
    <!-- 公告弹窗 -->
    <div v-if="showAnnouncement && announcementContent" class="announcement-modal-overlay">
      <div class="announcement-modal">
        <div class="announcement-header">
          <h3>📢 公告</h3>
          <button class="close-btn" @click="closeAnnouncement('session')">✕</button>
        </div>
        <div class="announcement-content" v-html="announcementContent"></div>
        <div class="announcement-footer">
          <button class="close-btn-primary" @click="closeAnnouncement('session')">本次关闭</button>
          <button class="close-btn-secondary" @click="closeAnnouncement('day')">本日关闭</button>
        </div>
      </div>
    </div>
    
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
    const announcementContent = ref('')
    const showAnnouncement = ref(true)
    
    // 检查是否需要显示公告
    const checkAnnouncementDisplay = () => {
      // 检查是否设置了本日关闭
      const dayCloseDate = localStorage.getItem('announcementDayClose');
      if (dayCloseDate) {
        const today = new Date().toDateString();
        if (dayCloseDate === today) {
          showAnnouncement.value = false;
          return;
        }
      }
      
      // 检查是否设置了本次关闭
      const sessionClose = sessionStorage.getItem('announcementSessionClose');
      if (sessionClose === 'true') {
        showAnnouncement.value = false;
      }
    }
    
    // 加载公告内容
    const loadAnnouncement = async () => {
      try {
        // 先检查是否需要显示公告
        checkAnnouncementDisplay();
        
        if (showAnnouncement.value) {
          const response = await fetch('/data/1.md');
          const markdownContent = await response.text();
          // 简单的Markdown转HTML处理
          let htmlContent = markdownContent
            .replace(/#{1,6}\s+([^\n]+)/g, (match, text, level) => {
              const hLevel = match.match(/#/g).length;
              return `<h${hLevel}>${text}</h${hLevel}>`;
            })
            .replace(/\*\*([^*]+)\*\*/g, '<strong>$1</strong>')
            .replace(/\*([^*]+)\*/g, '<em>$1</em>')
            .replace(/-\s+([^\n]+)/g, '<li>$1</li>')
            .replace(/(<li>.*?<\/li>)/gs, '<ul>$1</ul>')
            .replace(/\n/g, '<br>');
          announcementContent.value = htmlContent;
        }
      } catch (error) {
        console.error('Failed to load announcement:', error);
      }
    }
    
    // 关闭公告
    const closeAnnouncement = (type) => {
      showAnnouncement.value = false;
      
      if (type === 'session') {
        // 本次关闭，使用sessionStorage
        sessionStorage.setItem('announcementSessionClose', 'true');
      } else if (type === 'day') {
        // 本日关闭，使用localStorage存储日期
        const today = new Date().toDateString();
        localStorage.setItem('announcementDayClose', today);
      }
    }

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
      // 加载公告内容
      await loadAnnouncement();
      
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
    });
    
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
      toggleItem,
      announcementContent,
      showAnnouncement,
      closeAnnouncement
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
  
  /* 公告弹窗样式 */
  .announcement-modal-overlay {
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background-color: rgba(0, 0, 0, 0.5);
    display: flex;
    justify-content: center;
    align-items: center;
    z-index: 1000;
  }
  
  .announcement-modal {
    background: white;
    border-radius: 16px;
    width: 90%;
    max-width: 600px;
    max-height: 80vh;
    display: flex;
    flex-direction: column;
    box-shadow: 0 10px 30px rgba(0, 0, 0, 0.3);
    animation: slideIn 0.3s ease-out;
  }
  
  @keyframes slideIn {
    from {
      opacity: 0;
      transform: translateY(-20px);
    }
    to {
      opacity: 1;
      transform: translateY(0);
    }
  }
  
  .announcement-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 20px 20px 0;
    margin-bottom: 15px;
    border-bottom: 1px solid #eee;
    padding-bottom: 10px;
  }
  
  .announcement-header h3 {
    margin: 0;
    color: #333;
    font-size: 1.3em;
  }
  
  .close-btn {
    background: none;
    border: none;
    font-size: 1.5em;
    cursor: pointer;
    color: #999;
    padding: 0;
    width: 30px;
    height: 30px;
    display: flex;
    align-items: center;
    justify-content: center;
    border-radius: 50%;
    transition: all 0.2s;
  }
  
  .close-btn:hover {
    background-color: #f0f0f0;
    color: #333;
  }
  
  .announcement-footer {
    display: flex;
    justify-content: flex-end;
    gap: 10px;
    padding: 20px;
    border-top: 1px solid #eee;
    margin-top: auto;
  }
  
  .close-btn-primary, .close-btn-secondary {
    padding: 8px 20px;
    border-radius: 6px;
    border: none;
    cursor: pointer;
    font-size: 14px;
    font-weight: 500;
    transition: all 0.2s;
  }
  
  .close-btn-primary {
    background-color: #646cff;
    color: white;
  }
  
  .close-btn-primary:hover {
    background-color: #535bf2;
  }
  
  .close-btn-secondary {
    background-color: #f0f0f0;
    color: #333;
  }
  
  .close-btn-secondary:hover {
    background-color: #e0e0e0;
  }
  
  .announcement-content {
    line-height: 1.8;
    color: #444;
    padding: 0 20px;
    overflow-y: auto;
    max-height: calc(80vh - 120px);
  }
  
  .announcement-content h1, .announcement-content h2, .announcement-content h3,
  .announcement-content h4, .announcement-content h5, .announcement-content h6 {
    margin: 15px 0 10px 0;
    color: #333;
  }
  
  .announcement-content strong {
    font-weight: bold;
    color: #222;
  }
  
  .announcement-content em {
    font-style: italic;
  }
  
  .announcement-content ul {
    margin: 10px 0;
    padding-left: 20px;
  }
  
  .announcement-content li {
    margin: 5px 0;
  }
  
  .announcement-content br {
    margin: 5px 0;
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