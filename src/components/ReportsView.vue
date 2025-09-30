<template>
  <div class="reports-view">
    <h1>Отчетность и аналитика</h1>
    
    <div class="stats-grid">
      <div class="stat-card">
        <h3>Всего дефектов</h3>
        <div class="stat-number">{{ totalDefects }}</div>
      </div>
      <div class="stat-card">
        <h3>Новые</h3>
        <div class="stat-number new">{{ statusCounts.new }}</div>
      </div>
      <div class="stat-card">
        <h3>В работе</h3>
        <div class="stat-number in-progress">{{ statusCounts.in_progress }}</div>
      </div>
      <div class="stat-card">
        <h3>Решены</h3>
        <div class="stat-number resolved">{{ statusCounts.resolved }}</div>
      </div>
      <div class="stat-card">
        <h3>Закрыты</h3>
        <div class="stat-number closed">{{ statusCounts.closed }}</div>
      </div>
    </div>

    <div class="priority-stats">
      <h3>Распределение по приоритетам</h3>
      <div class="priority-bars">
        <div v-for="(count, priority) in priorityCounts" :key="priority" class="priority-bar">
          <span class="priority-label">{{ getPriorityText(priority) }}</span>
          <div class="bar-container">
            <div 
              :class="['bar', priority]" 
              :style="{ width: getPercentage(priority) + '%' }"
            ></div>
          </div>
          <span class="count">{{ count }}</span>
        </div>
      </div>
    </div>

    <button @click="exportToCSV" class="btn-export">Экспорт в CSV</button>
  </div>
</template>

<script>
import { ref, onMounted } from 'vue'

export default {
  name: 'ReportsView',
  props: {
    user: {
      type: Object,
      required: true
    }
  },
  setup() {
    const defects = ref([])
    const totalDefects = ref(0)
    const statusCounts = ref({
      new: 0,
      in_progress: 0,
      resolved: 0,
      closed: 0
    })
    const priorityCounts = ref({
      low: 0,
      medium: 0,
      high: 0
    })

    const loadData = () => {
      const savedDefects = localStorage.getItem('defects')
      if (savedDefects) {
        defects.value = JSON.parse(savedDefects)
        calculateStats()
      }
    }

    const calculateStats = () => {
      totalDefects.value = defects.value.length
      
      statusCounts.value = { new: 0, in_progress: 0, resolved: 0, closed: 0 }
      priorityCounts.value = { low: 0, medium: 0, high: 0 }

      defects.value.forEach(defect => {
        statusCounts.value[defect.status]++
        priorityCounts.value[defect.priority]++
      })
    }

    const getPriorityText = (priority) => {
      const texts = {
        low: 'Низкий',
        medium: 'Средний',
        high: 'Высокий'
      }
      return texts[priority] || priority
    }

    const getPercentage = (priority) => {
      const total = totalDefects.value
      if (total === 0) return 0
      return (priorityCounts.value[priority] / total) * 100
    }

    const exportToCSV = () => {
      const headers = ['ID', 'Название', 'Приоритет', 'Статус', 'Проект', 'Создал', 'Дата создания']
      const csvData = defects.value.map(defect => [
        defect.id,
        `"${defect.title}"`,
        getPriorityText(defect.priority),
        defect.status,
        defect.projectId,
        defect.createdBy,
        new Date(defect.createdAt).toLocaleDateString('ru-RU')
      ])

      const csvContent = [headers, ...csvData]
        .map(row => row.join(','))
        .join('\n')

      const blob = new Blob([csvContent], { type: 'text/csv;charset=utf-8;' })
      const link = document.createElement('a')
      const url = URL.createObjectURL(blob)
      
      link.setAttribute('href', url)
      link.setAttribute('download', 'defects_report.csv')
      link.style.visibility = 'hidden'
      
      document.body.appendChild(link)
      link.click()
      document.body.removeChild(link)
    }

    onMounted(() => {
      loadData()
    })

    return {
      totalDefects,
      statusCounts,
      priorityCounts,
      getPriorityText,
      getPercentage,
      exportToCSV
    }
  }
}
</script>

<style scoped>
.stats-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 1.5rem;
  margin: 2rem 0;
}

.stat-card {
  background: white;
  padding: 1.5rem;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
  text-align: center;
}

.stat-card h3 {
  margin: 0 0 1rem 0;
  color: #666;
  font-size: 0.9rem;
  text-transform: uppercase;
}

.stat-number {
  font-size: 2.5rem;
  font-weight: bold;
  color: #1890ff;
}

.stat-number.new { color: #faad14; }
.stat-number.in-progress { color: #1890ff; }
.stat-number.resolved { color: #52c41a; }
.stat-number.closed { color: #722ed1; }

.priority-stats {
  background: white;
  padding: 2rem;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
  margin: 2rem 0;
}

.priority-stats h3 {
  margin: 0 0 1.5rem 0;
  color: #333;
}

.priority-bars {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.priority-bar {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.priority-label {
  width: 80px;
  font-weight: bold;
}

.bar-container {
  flex: 1;
  height: 20px;
  background: #f0f0f0;
  border-radius: 10px;
  overflow: hidden;
}

.bar {
  height: 100%;
  transition: width 0.3s ease;
}

.bar.low { background: #52c41a; }
.bar.medium { background: #faad14; }
.bar.high { background: #ff4d4f; }

.count {
  width: 40px;
  text-align: right;
  font-weight: bold;
}

.btn-export {
  background-color: #52c41a;
  color: white;
  border: none;
  padding: 0.75rem 1.5rem;
  border-radius: 4px;
  cursor: pointer;
  font-size: 1rem;
}

.btn-export:hover {
  background-color: #73d13d;
}
</style>