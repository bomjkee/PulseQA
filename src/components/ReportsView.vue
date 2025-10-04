<template>
  <div class="reports-view">
    <header class="panel-header">
      <div>
        <h1>Аналитика и отчеты</h1>
        <p>Живой срез по дефектам за выбранный период: динамика, фокусные зоны, экспорт данных.</p>
      </div>
      <div class="controls">
        <label>
          <span>Период анализа</span>
          <select v-model="timeframe">
            <option value="14">14 дней</option>
            <option value="30">30 дней</option>
            <option value="90">90 дней</option>
            <option value="all">Вся история</option>
          </select>
        </label>
        <button class="ghost" @click="copySummary">Копировать сводку</button>
        <button class="primary" @click="exportData('csv')">CSV</button>
        <button class="primary" @click="exportData('json')">JSON</button>
      </div>
    </header>

    <section class="summary-grid">
      <article class="card">
        <h3>Всего дефектов</h3>
        <p class="number">{{ filteredDefects.length }}</p>
        <span class="caption">{{ totalCaption }}</span>
      </article>
      <article v-for="status in statusMetrics" :key="status.key" class="card">
        <h3>{{ status.label }}</h3>
        <p class="number">{{ status.count }}</p>
        <span :class="['trend', status.trendClass]">{{ status.trend }}</span>
      </article>
      <article class="card highlight">
        <h3>Средний темп закрытия</h3>
        <p class="number">{{ closingRate }}%</p>
        <span class="caption">За последние {{ timeframeLabel }}</span>
      </article>
    </section>

    <section class="charts">
      <div class="card">
        <header>
          <h3>Распределение по статусам</h3>
        </header>
        <ul class="bars">
          <li v-for="row in statusDistribution" :key="row.key">
            <span class="label">{{ row.label }}</span>
            <div class="bar-track">
              <div class="bar-fill" :style="{ width: row.percent + '%' }"></div>
            </div>
            <span class="value">{{ row.count }}</span>
          </li>
        </ul>
      </div>

      <div class="card">
        <header>
          <h3>Приоритеты</h3>
        </header>
        <ul class="chips">
          <li v-for="row in priorityDistribution" :key="row.key">
            <span :class="['dot', row.key]"></span>
            <span>{{ row.label }}</span>
            <strong>{{ row.count }}</strong>
          </li>
        </ul>
      </div>

      <div class="card span-2">
        <header>
          <h3>Динамика за неделю</h3>
          <span>{{ weeklyMetrics.length }} недель</span>
        </header>
        <div class="timeline">
          <div v-for="week in weeklyMetrics" :key="week.label" class="timeline-column">
            <span class="week-label">{{ week.label }}</span>
            <div class="stack">
              <div class="stack-bar new" :style="{ height: week.new + '%' }" :title="`Новые: ${week.newCount}`"></div>
              <div class="stack-bar closed" :style="{ height: week.closed + '%' }" :title="`Закрытые: ${week.closedCount}`"></div>
            </div>
            <span class="count">{{ week.total }}</span>
          </div>
        </div>
      </div>

      <div class="card span-2">
        <header>
          <h3>Проекты с рисками</h3>
        </header>
        <ul class="risk-list">
          <li v-for="project in riskProjects" :key="project.id">
            <div>
              <p class="title">{{ project.name }}</p>
              <p class="meta">Просрочено: {{ project.overdue }} · В работе: {{ project.active }}</p>
            </div>
            <span class="badge alert">{{ project.health }}</span>
          </li>
          <li v-if="!riskProjects.length" class="empty">Нет проектов с просроченными задачами</li>
        </ul>
      </div>
    </section>
  </div>
</template>

<script>
import { computed, onMounted, ref } from 'vue'

export default {
  name: 'ReportsView',
  emits: ['sync'],
  props: {
    user: {
      type: Object,
      required: true
    }
  },
  setup(props, { emit }) {
    const defects = ref([])
    const projects = ref([])
    const timeframe = ref('30')

    const loadData = () => {
      defects.value = JSON.parse(localStorage.getItem('defects') || '[]')
      projects.value = JSON.parse(localStorage.getItem('projects') || '[]')
    }

    const timeframeLabel = computed(() => {
      if (timeframe.value === 'all') return 'всю историю'
      return `${timeframe.value} дней`
    })

    const filteredDefects = computed(() => {
      if (timeframe.value === 'all') return defects.value
      const days = Number(timeframe.value)
      const threshold = new Date()
      threshold.setDate(threshold.getDate() - days)
      return defects.value.filter(defect => new Date(defect.createdAt) >= threshold)
    })

    const totalCaption = computed(() => {
      if (!filteredDefects.value.length) return 'Данные не найдены'
      if (timeframe.value === 'all') return 'С начала ведения системы'
      return `Создано/актуализировано за ${timeframeLabel.value}`
    })

    const statusMetrics = computed(() => {
      const counts = { new: 0, in_progress: 0, resolved: 0, closed: 0 }
      filteredDefects.value.forEach(defect => {
        counts[defect.status] = (counts[defect.status] || 0) + 1
      })

      const previousPeriod = () => {
        if (timeframe.value === 'all') return []
        const days = Number(timeframe.value)
        const start = new Date()
        const end = new Date()
        start.setDate(start.getDate() - days * 2)
        end.setDate(end.getDate() - days)
        return defects.value.filter(defect => {
          const created = new Date(defect.createdAt)
          return created >= start && created < end
        })
      }

      const previous = previousPeriod()
      const prevCounts = previous.reduce((acc, defect) => {
        acc[defect.status] = (acc[defect.status] || 0) + 1
        return acc
      }, {})

      const statusLabels = {
        new: 'Новые',
        in_progress: 'В работе',
        resolved: 'Решены',
        closed: 'Закрыты'
      }

      const formatTrend = (current, previousValue) => {
        if (timeframe.value === 'all') return 'История'
        const diff = current - (previousValue || 0)
        if (diff > 0) return `+${diff} vs прошлый период`
        if (diff < 0) return `${diff} vs прошлый период`
        return 'Без изменений'
      }

      const trendClass = (current, previousValue) => {
        if (timeframe.value === 'all') return 'neutral'
        const diff = current - (previousValue || 0)
        if (diff > 0 && (previousValue || 0) !== 0) return 'up'
        if (diff < 0) return 'down'
        return 'neutral'
      }

      return Object.keys(statusLabels).map(key => ({
        key,
        label: statusLabels[key],
        count: counts[key] || 0,
        trend: formatTrend(counts[key] || 0, prevCounts[key] || 0),
        trendClass: trendClass(counts[key] || 0, prevCounts[key] || 0)
      }))
    })

    const closingRate = computed(() => {
      const resolved = filteredDefects.value.filter(defect => ['resolved', 'closed'].includes(defect.status)).length
      if (!filteredDefects.value.length) return 0
      return Math.round((resolved / filteredDefects.value.length) * 100)
    })

    const statusDistribution = computed(() => {
      const total = filteredDefects.value.length || 1
      const labels = {
        new: 'Новые',
        in_progress: 'В работе',
        resolved: 'Решены',
        closed: 'Закрыты'
      }

      return Object.keys(labels).map(key => {
        const count = filteredDefects.value.filter(defect => defect.status === key).length
        return {
          key,
          label: labels[key],
          count,
          percent: Math.round((count / total) * 100)
        }
      })
    })

    const priorityDistribution = computed(() => {
      const labels = {
        high: 'Высокий приоритет',
        medium: 'Средний приоритет',
        low: 'Низкий приоритет'
      }
      return Object.keys(labels).map(key => ({
        key,
        label: labels[key],
        count: filteredDefects.value.filter(defect => defect.priority === key).length
      }))
    })

    const weeklyMetrics = computed(() => {
      const buckets = new Map()
      filteredDefects.value.forEach(defect => {
        const date = new Date(defect.createdAt)
        const monday = new Date(date)
        const day = monday.getDay() || 7
        monday.setHours(0, 0, 0, 0)
        monday.setDate(monday.getDate() - day + 1)
        const key = monday.toISOString()
        const bucket = buckets.get(key) || { new: 0, closed: 0, total: 0 }
        bucket.total += 1
        if (defect.status === 'new') bucket.new += 1
        if (['resolved', 'closed'].includes(defect.status)) bucket.closed += 1
        buckets.set(key, bucket)
      })

      const entries = Array.from(buckets.entries())
        .sort((a, b) => new Date(a[0]) - new Date(b[0]))
        .slice(-6)

      return entries.map(([key, value]) => {
        const date = new Date(key)
        const label = new Intl.DateTimeFormat('ru-RU', { day: '2-digit', month: '2-digit' }).format(date)
        const max = Math.max(value.total, 1)
        return {
          label,
          total: value.total,
          new: Math.round((value.new / max) * 100),
          closed: Math.round((value.closed / max) * 100),
          newCount: value.new,
          closedCount: value.closed
        }
      })
    })

    const riskProjects = computed(() => {
      return projects.value
        .map(project => {
          const projectDefects = filteredDefects.value.filter(defect => defect.projectId === project.id)
          const overdue = projectDefects.filter(defect => defect.deadline && new Date(defect.deadline) < new Date() && !['resolved', 'closed'].includes(defect.status))
          const active = projectDefects.filter(defect => ['new', 'in_progress'].includes(defect.status))
          return {
            id: project.id,
            name: project.name,
            overdue: overdue.length,
            active: active.length,
            health: overdue.length ? 'Требует внимания' : 'Под контролем'
          }
        })
        .filter(project => project.overdue > 0)
        .sort((a, b) => b.overdue - a.overdue)
        .slice(0, 4)
    })

    const exportData = (type) => {
      const dataset = filteredDefects.value.map(defect => ({
        id: defect.id,
        title: defect.title,
        priority: defect.priority,
        status: defect.status,
        projectId: defect.projectId,
        assignee: defect.assignee,
        deadline: defect.deadline,
        createdBy: defect.createdBy,
        createdAt: defect.createdAt
      }))

      if (type === 'json') {
        const blob = new Blob([JSON.stringify(dataset, null, 2)], { type: 'application/json' })
        triggerDownload(blob, 'defects_report.json')
        return
      }

      const headers = ['ID', 'Название', 'Приоритет', 'Статус', 'Проект', 'Ответственный', 'Дедлайн', 'Создал', 'Дата создания']
      const projectMap = new Map(projects.value.map(project => [project.id, project.name]))

      const rows = dataset.map(defect => [
        defect.id,
        `"${defect.title}"`,
        defect.priority,
        defect.status,
        projectMap.get(defect.projectId) || '—',
        defect.assignee || '—',
        defect.deadline || '—',
        defect.createdBy || '—',
        new Date(defect.createdAt).toLocaleString('ru-RU')
      ])

      const csvContent = [headers, ...rows]
        .map(row => row.join(','))
        .join('\n')

      const blob = new Blob([csvContent], { type: 'text/csv;charset=utf-8;' })
      triggerDownload(blob, 'defects_report.csv')
    }

    const triggerDownload = (blob, filename) => {
      const link = document.createElement('a')
      const url = URL.createObjectURL(blob)
      link.setAttribute('href', url)
      link.setAttribute('download', filename)
      link.style.visibility = 'hidden'
      document.body.appendChild(link)
      link.click()
      document.body.removeChild(link)
      URL.revokeObjectURL(url)
    }

    const copySummary = async () => {
      const lines = [
        `Сводка по качеству (период: ${timeframeLabel.value})`,
        `Всего дефектов: ${filteredDefects.value.length}`,
        ...statusMetrics.value.map(row => `${row.label}: ${row.count} (${row.trend})`),
        `Средний темп закрытия: ${closingRate.value}%`
      ]

      try {
        await navigator.clipboard.writeText(lines.join('\n'))
      } catch (error) {
        console.warn('Не удалось скопировать в буфер обмена', error)
      }
    }

    onMounted(() => {
      loadData()
      emit('sync')
    })

    return {
      timeframe,
      timeframeLabel,
      filteredDefects,
      totalCaption,
      statusMetrics,
      statusDistribution,
      priorityDistribution,
      weeklyMetrics,
      riskProjects,
      closingRate,
      exportData,
      copySummary
    }
  }
}
</script>

<style scoped>
.panel-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  gap: 2rem;
  margin-bottom: 2rem;
}

.panel-header h1 {
  font-size: 1.8rem;
  color: var(--text-primary);
}

.panel-header p {
  color: var(--text-muted);
  margin-top: 0.45rem;
}

.controls {
  display: flex;
  gap: 0.75rem;
  align-items: center;
}

.controls label {
  display: grid;
  gap: 0.3rem;
  font-size: 0.8rem;
  text-transform: uppercase;
  letter-spacing: 0.3px;
  color: var(--text-light);
}

.controls select {
  border-radius: 12px;
  border: 1px solid var(--input-border);
  background: var(--input-bg);
  color: var(--input-text);
  padding: 0.5rem 0.9rem;
  transition: var(--transition);
}

.ghost,
.primary {
  border-radius: 14px;
  padding: 0.6rem 1rem;
  border: 1px solid var(--button-ghost-border);
  background: transparent;
  color: var(--text-light);
  cursor: pointer;
  transition: var(--transition);
}

.ghost:hover {
  background: var(--button-ghost-hover);
}

.primary {
  background: var(--button-primary-bg);
  border: none;
  color: var(--button-primary-text);
  font-weight: 700;
}

.summary-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(220px, 1fr));
  gap: 1.5rem;
  margin-bottom: 2rem;
}

.card {
  background: var(--card-bg);
  border-radius: 24px;
  border: 1px solid var(--surface-border);
  padding: 1.6rem;
  display: grid;
  gap: 0.6rem;
  transition: var(--transition);
}

.card header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.card h3 {
  font-size: 0.95rem;
  text-transform: uppercase;
  letter-spacing: 0.3px;
  color: var(--text-light);
}

.card .number {
  font-size: 2.4rem;
  font-weight: 700;
  color: var(--text-primary);
}

.card .caption {
  color: var(--text-muted);
  font-size: 0.85rem;
}

.card.highlight {
  background: var(--card-highlight-bg);
}

.trend {
  font-size: 0.85rem;
}

.trend.up { color: var(--accent-success); }
.trend.down { color: var(--accent-danger); }
.trend.neutral { color: var(--text-muted); }

.charts {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(260px, 1fr));
  gap: 1.5rem;
}

.span-2 {
  grid-column: span 2;
}

.bars {
  list-style: none;
  display: grid;
  gap: 1rem;
}

.bars li {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.bars .label {
  width: 120px;
  color: var(--text-light);
}

.bar-track {
  flex: 1;
  height: 10px;
  border-radius: 999px;
  background: var(--progress-bar-bg);
  overflow: hidden;
}

.bar-fill {
  height: 100%;
  background: var(--progress-bar-fill);
  transition: width 0.3s ease;
}

.bars .value {
  width: 40px;
  text-align: right;
  color: var(--text-light);
}

.chips {
  list-style: none;
  display: grid;
  gap: 0.8rem;
}

.chips li {
  display: flex;
  align-items: center;
  gap: 0.7rem;
  background: var(--chip-bg);
  padding: 0.7rem 1rem;
  border-radius: 14px;
  border: 1px solid var(--chip-border);
  transition: var(--transition);
}

.chips .dot {
  width: 10px;
  height: 10px;
  border-radius: 50%;
}

.chips .dot.high { background: var(--accent-danger); }
.chips .dot.medium { background: var(--accent-warning); }
.chips .dot.low { background: var(--accent-success); }

.chips strong {
  margin-left: auto;
  color: var(--text-light);
}

.timeline {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(80px, 1fr));
  gap: 1rem;
  align-items: end;
}

.timeline-column {
  display: grid;
  gap: 0.4rem;
  justify-items: center;
}

.week-label {
  font-size: 0.75rem;
  color: var(--text-muted);
}

.stack {
  width: 100%;
  height: 140px;
  border-radius: 14px;
  background: var(--chip-bg);
  border: 1px solid var(--chip-border);
  display: flex;
  flex-direction: column-reverse;
  overflow: hidden;
  transition: var(--transition);
}

.stack-bar {
  width: 100%;
}

.stack-bar.new { background: var(--accent-primary); opacity: 0.5; }
.stack-bar.closed { background: var(--accent-success); opacity: 0.5; }

.timeline .count {
  font-size: 0.85rem;
  color: var(--text-light);
}

.risk-list {
  list-style: none;
  display: grid;
  gap: 0.9rem;
}

.risk-list li {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 1.2rem;
  background: var(--chip-bg);
  border-radius: 16px;
  padding: 0.9rem 1.2rem;
  border: 1px solid var(--chip-border);
  transition: var(--transition);
}

.risk-list .title {
  color: var(--text-primary);
  font-weight: 600;
}

.risk-list .meta {
  color: var(--text-muted);
  font-size: 0.85rem;
}

.risk-list .empty {
  justify-content: center;
  color: var(--text-muted);
}

.badge.alert {
  background: var(--badge-alert-bg);
  color: var(--badge-alert-text);
  border-radius: 999px;
  padding: 0.3rem 0.8rem;
  text-transform: uppercase;
  font-size: 0.75rem;
}

@media (max-width: 960px) {
  .controls {
    flex-wrap: wrap;
  }

  .charts {
    grid-template-columns: 1fr;
  }

  .span-2 {
    grid-column: span 1;
  }
}

@media (max-width: 480px) {
  .panel-header {
    flex-direction: column;
    align-items: flex-start;
    padding: 0.7rem 0.3rem;
  }
  .summary-grid, .charts {
    grid-template-columns: 1fr;
    gap: 0.5rem;
    padding: 0.2rem;
  }
  .card {
    padding: 0.7rem 0.4rem;
    font-size: 0.97rem;
  }
  .timeline {
    gap: 0.5rem;
  }
  .risk-list li {
    padding: 0.5rem 0.2rem;
    font-size: 0.95rem;
  }
}

</style>
