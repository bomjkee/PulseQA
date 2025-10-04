<template>
  <div class="defects-view">
    <header class="panel-header">
      <div>
        <h1>Центр управления дефектами</h1>
        <p>Планируйте ремонт, назначайте ответственных и отслеживайте динамику устранения.</p>
      </div>
      <div class="header-actions">
        <button class="ghost" @click="filtersOpen = !filtersOpen">
          {{ filtersOpen ? 'Скрыть фильтры' : 'Показать фильтры' }}
        </button>
        <button class="ghost" @click="toggleDisplayMode">
          {{ displayMode === 'list' ? 'Вид канбан' : 'Список' }}
        </button>
        <button class="primary" @click="openNewDefect">Новый дефект</button>
      </div>
    </header>

    <section class="quick-stats">
      <article v-for="stat in statusSummary" :key="stat.key" class="stat">
        <span class="label">{{ stat.label }}</span>
        <strong>{{ stat.value }}</strong>
        <small>{{ stat.hint }}</small>
      </article>
      <article class="stat accent">
        <span class="label">Просроченные</span>
        <strong>{{ overdueCount }}</strong>
        <small>Дедлайн истек или сегодня</small>
      </article>
    </section>

    <transition name="fade">
      <section v-if="filtersOpen" class="filters">
        <div class="field">
          <label>Поиск</label>
          <input v-model="filters.search" type="search" placeholder="Название, описание, теги или исполнитель">
        </div>
        <div class="field">
          <label>Статусы</label>
          <div class="chips">
            <button
              v-for="status in statusOptions"
              :key="status.value"
              :class="['chip', { active: filters.status.includes(status.value) }]"
              type="button"
              @click="toggleFilterValue('status', status.value)"
            >
              {{ status.label }}
            </button>
          </div>
        </div>
        <div class="field">
          <label>Приоритет</label>
          <div class="chips">
            <button
              v-for="priority in priorityOptions"
              :key="priority.value"
              :class="['chip', { active: filters.priority.includes(priority.value) }]"
              type="button"
              @click="toggleFilterValue('priority', priority.value)"
            >
              {{ priority.label }}
            </button>
          </div>
        </div>
        <div class="field">
          <label>Проект</label>
          <select v-model="filters.project">
            <option value="all">Все проекты</option>
            <option v-for="project in projects" :key="project.id" :value="project.id">
              {{ project.name }}
            </option>
          </select>
        </div>
        <div class="field">
          <label>Ответственный</label>
          <select v-model="filters.assignee">
            <option value="all">Любой</option>
            <option v-for="member in teamMembers" :key="member.id" :value="member.name">
              {{ member.name }}
            </option>
          </select>
        </div>
        <div class="field inline">
          <label>
            <input v-model="filters.overdueOnly" type="checkbox">
            Показывать только просроченные
          </label>
          <label>
            <span>Сортировка</span>
            <select v-model="sortOption">
              <option value="newest">Сначала новые</option>
              <option value="deadline">По дедлайну</option>
              <option value="priority">По приоритету</option>
            </select>
          </label>
        </div>
      </section>
    </transition>

    <transition name="fade">
      <div v-if="showAddForm" class="modal">
        <div class="modal-content">
          <header>
            <h3>Новый дефект</h3>
            <button type="button" class="icon" @click="closeNewDefect">×</button>
          </header>
          <form @submit.prevent="addDefect" class="modal-form">
            <label>
              <span>Название</span>
              <input v-model="newDefect.title" required placeholder="Например, Трещина в панелях на 3 этаже">
            </label>
            <label>
              <span>Описание</span>
              <textarea v-model="newDefect.description" rows="3" placeholder="Кратко опишите проблему и ожидаемый результат"></textarea>
            </label>
            <div class="grid">
              <label>
                <span>Проект</span>
                <select v-model="newDefect.projectId" required>
                  <option value="" disabled>Выберите проект</option>
                  <option v-for="project in projects" :key="project.id" :value="project.id">
                    {{ project.name }}
                  </option>
                </select>
              </label>
              <label>
                <span>Приоритет</span>
                <select v-model="newDefect.priority">
                  <option value="low">Низкий</option>
                  <option value="medium">Средний</option>
                  <option value="high">Высокий</option>
                </select>
              </label>
              <label>
                <span>Влияние на безопасность</span>
                <select v-model="newDefect.impact">
                  <option value="low">Незначительное</option>
                  <option value="medium">Среднее</option>
                  <option value="critical">Критическое</option>
                </select>
              </label>
              <label>
                <span>Дедлайн</span>
                <input v-model="newDefect.deadline" type="date">
              </label>
            </div>
            <div class="grid">
              <label>
                <span>Ответственный</span>
                <select v-model="newDefect.assignee">
                  <option value="">Назначить позже</option>
                  <option v-for="member in teamMembers" :key="member.id" :value="member.name">
                    {{ member.name }} ({{ member.role }})
                  </option>
                </select>
              </label>
              <label>
                <span>Теги</span>
                <input v-model="newDefect.tags" placeholder="Например: фасад, безопасность">
              </label>
            </div>
            <label>
              <span>Дополнительная информация</span>
              <textarea v-model="newDefect.notes" rows="2" placeholder="План работ, доступы, материалы"></textarea>
            </label>
            <footer>
              <button type="button" class="ghost" @click="closeNewDefect">Отмена</button>
              <button type="submit" class="primary">Сохранить</button>
            </footer>
          </form>
        </div>
      </div>
    </transition>

    <section v-if="!filteredDefects.length" class="empty-state">
      <h3>Ничего не найдено</h3>
      <p>Сбросьте фильтры или добавьте новый дефект.</p>
    </section>

    <section v-else-if="displayMode === 'list'" class="defects-list">
      <article v-for="defect in filteredDefects" :key="defect.id" class="defect-card">
        <header>
          <div>
            <h3>{{ defect.title }}</h3>
            <p class="meta">Создан {{ formatDate(defect.createdAt) }} · {{ defect.createdBy }}</p>
          </div>
          <div class="labels">
            <span :class="['badge', 'priority', defect.priority]">{{ getPriorityText(defect.priority) }}</span>
            <span :class="['badge', 'status', defect.status]">{{ getStatusText(defect.status) }}</span>
            <span v-if="defect.impact" :class="['badge', 'impact', defect.impact]">{{ impactLabels[defect.impact] }}</span>
          </div>
        </header>

        <p class="description">{{ defect.description || 'Описание не добавлено' }}</p>

        <ul v-if="defect.tags?.length" class="tags">
          <li v-for="tag in defect.tags" :key="tag">#{{ tag }}</li>
        </ul>

        <dl class="details">
          <div>
            <dt>Проект</dt>
            <dd>{{ getProjectName(defect.projectId) }}</dd>
          </div>
          <div>
            <dt>Ответственный</dt>
            <dd>{{ defect.assignee || 'Не назначен' }}</dd>
          </div>
          <div>
            <dt>Дедлайн</dt>
            <dd>{{ formatDeadline(defect.deadline) }}</dd>
          </div>
        </dl>

        <section v-if="defect.history?.length" class="history">
          <h4>История</h4>
          <ul>
            <li v-for="entry in defect.history.slice(-3).reverse()" :key="entry.at">
              <span>{{ formatDateTime(entry.at) }}</span>
              <span>{{ entry.user }} → {{ getStatusText(entry.status) }}</span>
            </li>
          </ul>
        </section>

        <footer class="actions">
          <button
            v-for="transition in availableTransitions(defect.status)"
            :key="transition.status"
            class="ghost"
            @click="updateDefectStatus(defect.id, transition.status)"
          >
            {{ transition.label }}
          </button>
          <button
            v-if="canManage"
            class="ghost danger"
            @click="archiveDefect(defect.id)"
          >
            Архивировать
          </button>
        </footer>
      </article>
    </section>

    <section v-else class="kanban">
      <div v-for="column in kanbanColumns" :key="column.status" class="kanban-column">
        <header>
          <h3>{{ column.title }}</h3>
          <span>{{ column.items.length }}</span>
        </header>
        <div class="kanban-cards">
          <article v-for="item in column.items" :key="item.id" class="kanban-card">
            <p class="title">{{ item.title }}</p>
            <p class="meta">{{ getProjectName(item.projectId) }}</p>
            <p class="assignee">{{ item.assignee || 'Нет исполнителя' }}</p>
            <span :class="['badge', 'priority', item.priority]">{{ getPriorityText(item.priority) }}</span>
            <button
              v-for="transition in availableTransitions(item.status)"
              :key="transition.status"
              class="mini"
              @click="updateDefectStatus(item.id, transition.status)"
            >
              {{ transition.short }}
            </button>
          </article>
        </div>
      </div>
    </section>
  </div>
</template>

<script>
import { computed, onMounted, reactive, ref, watch } from 'vue'

export default {
  name: 'DefectsView',
  emits: ['sync'],
  props: {
    user: {
      type: Object,
      required: true
    }
  },
  setup(props, { emit }) {
    const showAddForm = ref(false)
    const filtersOpen = ref(false)
    const displayMode = ref('list')
    const defects = ref([])
    const projects = ref([])
    const sortOption = ref('newest')

    const teamMembers = [
      { id: 'qa', name: 'Анна Дмитриева', role: 'Инженер QA' },
      { id: 'supervisor', name: 'Николай Егоров', role: 'Прораб участка' },
      { id: 'architect', name: 'Мария Орлова', role: 'Архитектор' },
      { id: 'safety', name: 'Илья Савчук', role: 'Инженер по безопасности' }
    ]

    const newDefect = reactive({
      title: '',
      description: '',
      priority: 'medium',
      projectId: '',
      deadline: '',
      assignee: '',
      tags: '',
      notes: '',
      impact: 'medium'
    })

    const filters = reactive({
      search: '',
      status: ['new', 'in_progress', 'resolved', 'closed'],
      priority: [],
      project: 'all',
      assignee: 'all',
      overdueOnly: false
    })

    const statusOptions = [
      { value: 'new', label: 'Новые' },
      { value: 'in_progress', label: 'В работе' },
      { value: 'resolved', label: 'Решены' },
      { value: 'closed', label: 'Закрыты' }
    ]

    const priorityOptions = [
      { value: 'high', label: 'Высокий' },
      { value: 'medium', label: 'Средний' },
      { value: 'low', label: 'Низкий' }
    ]

    const impactLabels = {
      low: 'Незначительное влияние',
      medium: 'Среднее влияние',
      critical: 'Критическое влияние'
    }

    const priorityWeight = { high: 1, medium: 2, low: 3 }

    const canManage = computed(() => ['manager', 'admin'].includes(props.user.role))

    const normalizeDefect = (defect) => ({
      tags: [],
      notes: '',
      impact: 'medium',
      assignee: '',
      deadline: '',
      history: [],
      createdAt: new Date().toISOString(),
      ...defect
    })

    const loadData = () => {
      const savedDefects = JSON.parse(localStorage.getItem('defects') || '[]')
      const savedProjects = JSON.parse(localStorage.getItem('projects') || '[]')

      defects.value = savedDefects.map(normalizeDefect)

      projects.value = savedProjects.length
        ? savedProjects
        : [
            { id: 1, name: 'ЖК «Северный меридиан»', address: 'ул. Ленина, 1' },
            { id: 2, name: 'Бизнес-центр «Восток»', address: 'пр. Мира, 15' },
            { id: 3, name: 'Школа №45', address: 'ул. Школьная, 23' }
          ]

      if (!savedProjects.length) {
        localStorage.setItem('projects', JSON.stringify(projects.value))
      }

      if (!newDefect.projectId && projects.value.length) {
        newDefect.projectId = projects.value[0].id
      }
    }

    const openNewDefect = () => {
      showAddForm.value = true
    }

    const closeNewDefect = () => {
      showAddForm.value = false
      Object.assign(newDefect, {
        title: '',
        description: '',
        priority: 'medium',
        projectId: projects.value[0]?.id || '',
        deadline: '',
        assignee: '',
        tags: '',
        notes: '',
        impact: 'medium'
      })
    }

    const addDefect = () => {
      const tags = newDefect.tags
        .split(',')
        .map(tag => tag.trim())
        .filter(Boolean)

      const defect = normalizeDefect({
        id: Date.now(),
        title: newDefect.title,
        description: newDefect.description,
        priority: newDefect.priority,
        projectId: newDefect.projectId,
        deadline: newDefect.deadline,
        assignee: newDefect.assignee,
        tags,
        notes: newDefect.notes,
        impact: newDefect.impact,
        status: 'new',
        createdBy: props.user.name,
        createdAt: new Date().toISOString(),
        history: [
          {
            status: 'new',
            user: props.user.name,
            at: new Date().toISOString()
          }
        ]
      })

      defects.value.unshift(defect)
      closeNewDefect()
    }

    const updateDefectStatus = (defectId, newStatus) => {
      const defect = defects.value.find(d => d.id === defectId)
      if (!defect) return

      defect.status = newStatus
      defect.history = defect.history || []
      defect.history.push({
        status: newStatus,
        user: props.user.name,
        at: new Date().toISOString()
      })
    }

    const archiveDefect = (defectId) => {
      defects.value = defects.value.filter(defect => defect.id !== defectId)
    }

    const toggleDisplayMode = () => {
      displayMode.value = displayMode.value === 'list' ? 'board' : 'list'
    }

    const toggleFilterValue = (field, value) => {
      const list = filters[field]
      const index = list.indexOf(value)
      if (index !== -1) {
        list.splice(index, 1)
      } else {
        list.push(value)
      }
    }

    const filteredDefects = computed(() => {
      const search = filters.search.trim().toLowerCase()
      const today = new Date().setHours(0, 0, 0, 0)

      const result = defects.value.filter(defect => {
        const matchesSearch = !search || [
          defect.title,
          defect.description,
          defect.assignee,
          ...(defect.tags || [])
        ]
          .filter(Boolean)
          .some(field => field.toLowerCase().includes(search))

        const matchesStatus = !filters.status.length || filters.status.includes(defect.status)

        const matchesPriority = !filters.priority.length || filters.priority.includes(defect.priority)

        const matchesProject = filters.project === 'all' || defect.projectId === filters.project

        const matchesAssignee = filters.assignee === 'all' || defect.assignee === filters.assignee

        const matchesOverdue = !filters.overdueOnly || (
          defect.deadline &&
          new Date(defect.deadline).setHours(0, 0, 0, 0) <= today &&
          !['resolved', 'closed'].includes(defect.status)
        )

        return matchesSearch && matchesStatus && matchesPriority && matchesProject && matchesAssignee && matchesOverdue
      })

      const sorters = {
        newest: (a, b) => new Date(b.createdAt).getTime() - new Date(a.createdAt).getTime(),
        deadline: (a, b) => {
          if (!a.deadline && !b.deadline) return 0
          if (!a.deadline) return 1
          if (!b.deadline) return -1
          return new Date(a.deadline).getTime() - new Date(b.deadline).getTime()
        },
        priority: (a, b) => priorityWeight[a.priority] - priorityWeight[b.priority]
      }

      return result.sort(sorters[sortOption.value] || sorters.newest)
    })

    const kanbanColumns = computed(() => statusOptions.map(option => ({
      status: option.value,
      title: option.label,
      items: filteredDefects.value.filter(defect => defect.status === option.value)
    })))

    const statusSummary = computed(() => statusOptions.map(option => ({
      key: option.value,
      label: option.label,
      value: defects.value.filter(defect => defect.status === option.value).length,
      hint: `Последнее обновление: ${humanizeLastChange(option.value)}`
    })))

    const overdueCount = computed(() => {
      const today = new Date().setHours(0, 0, 0, 0)
      return defects.value.filter(defect => (
        defect.deadline &&
        new Date(defect.deadline).setHours(0, 0, 0, 0) <= today &&
        !['resolved', 'closed'].includes(defect.status)
      )).length
    })

    const humanizeLastChange = (status) => {
      const entries = defects.value
        .flatMap(defect => defect.history || [])
        .filter(entry => entry.status === status)
        .sort((a, b) => new Date(b.at).getTime() - new Date(a.at).getTime())

      if (!entries.length) return 'нет изменений'
      return formatDateTime(entries[0].at)
    }

    const availableTransitions = (status) => {
      const map = {
        new: [
          { status: 'in_progress', label: 'Отправить в работу', short: 'В работу' }
        ],
        in_progress: [
          { status: 'resolved', label: 'Пометить как решено', short: 'Решено' }
        ],
        resolved: [
          { status: 'closed', label: 'Закрыть дефект', short: 'Закр.' },
          { status: 'in_progress', label: 'Вернуть в работу', short: 'Назад' }
        ],
        closed: [
          { status: 'in_progress', label: 'Возобновить', short: 'Открыть' }
        ]
      }
      return map[status] || []
    }

    const getPriorityText = (priority) => {
      const texts = {
        low: 'Низкий',
        medium: 'Средний',
        high: 'Высокий'
      }
      return texts[priority] || priority
    }

    const getStatusText = (status) => {
      const texts = {
        new: 'Новый',
        in_progress: 'В работе',
        resolved: 'Решен',
        closed: 'Закрыт'
      }
      return texts[status] || status
    }

    const getProjectName = (projectId) => {
      const project = projects.value.find(p => p.id === projectId)
      return project ? project.name : 'Проект не найден'
    }

    const formatDate = (value) => {
      if (!value) return 'Сегодня'
      return new Intl.DateTimeFormat('ru-RU', {
        day: '2-digit',
        month: '2-digit'
      }).format(new Date(value))
    }

    const formatDateTime = (value) => {
      return new Intl.DateTimeFormat('ru-RU', {
        day: '2-digit',
        month: '2-digit',
        hour: '2-digit',
        minute: '2-digit'
      }).format(new Date(value))
    }

    const formatDeadline = (value) => {
      if (!value) return 'Не задан'
      const date = new Date(value)
      const today = new Date().setHours(0, 0, 0, 0)
      const diff = Math.ceil((date - today) / (1000 * 60 * 60 * 24))
      const formatted = new Intl.DateTimeFormat('ru-RU').format(date)

      if (diff < 0) return `${formatted} · просрочен`
      if (diff === 0) return `${formatted} · сегодня`
      return `${formatted} · через ${diff} д.`
    }

    watch(defects, (value) => {
      localStorage.setItem('defects', JSON.stringify(value))
      emit('sync')
    }, { deep: true })

    onMounted(() => {
      loadData()
    })

    return {
      teamMembers,
      filters,
      filtersOpen,
      showAddForm,
      displayMode,
      defects,
      projects,
      newDefect,
      statusOptions,
      priorityOptions,
      impactLabels,
      filteredDefects,
      statusSummary,
      overdueCount,
      sortOption,
      kanbanColumns,
      canManage,
      addDefect,
      openNewDefect,
      closeNewDefect,
      toggleFilterValue,
      toggleDisplayMode,
      updateDefectStatus,
      archiveDefect,
      availableTransitions,
      getPriorityText,
      getStatusText,
      getProjectName,
      formatDate,
      formatDateTime,
      formatDeadline
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
  margin-bottom: 2.5rem;
}

.panel-header h1 {
  font-size: 1.8rem;
  color: var(--text-primary);
}

.panel-header p {
  color: var(--text-muted);
  margin-top: 0.45rem;
}

.header-actions {
  display: flex;
  gap: 0.75rem;
}

.ghost,
.primary {
  border-radius: 14px;
  padding: 0.65rem 1.2rem;
  border: 1px solid var(--button-ghost-border);
  background: transparent;
  color: var(--text-light);
  cursor: pointer;
  transition: var(--transition);
}

.ghost:hover {
  background: var(--button-ghost-hover);
}

.ghost.danger {
  border-color: rgba(248, 113, 113, 0.4);
  color: var(--accent-danger);
}

.ghost.danger:hover {
  background: rgba(248, 113, 113, 0.15);
}

.primary {
  background: var(--button-primary-bg);
  border: none;
  color: var(--button-primary-text);
  font-weight: 700;
}

.primary:hover {
  transform: translateY(-1px);
  box-shadow: 0 12px 26px rgba(99, 102, 241, 0.35);
}

.quick-stats {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(180px, 1fr));
  gap: 1.2rem;
  margin-bottom: 2rem;
}

.stat {
  background: var(--card-bg);
  border-radius: 20px;
  border: 1px solid var(--card-border);
  padding: 1.2rem;
  display: grid;
  gap: 0.4rem;
  transition: var(--transition);
}

.stat.accent {
  background: var(--card-highlight-bg);
}

.stat .label {
  font-size: 0.8rem;
  letter-spacing: 0.4px;
  text-transform: uppercase;
  color: var(--text-light);
}

.stat strong {
  font-size: 1.8rem;
  color: var(--text-primary);
}

.stat small {
  color: var(--text-muted);
  font-size: 0.85rem;
}

.filters {
  background: var(--panel-bg);
  border: 1px solid var(--surface-border);
  border-radius: 20px;
  padding: 1.8rem;
  display: grid;
  gap: 1.4rem;
  margin-bottom: 2rem;
  transition: var(--transition);
}

.filters .field {
  display: grid;
  gap: 0.6rem;
}

.filters label {
  font-size: 0.85rem;
  color: var(--text-light);
}

.filters input,
.filters select {
  border-radius: 12px;
  border: 1px solid var(--input-border);
  background: var(--input-bg);
  color: var(--input-text);
  padding: 0.65rem 0.9rem;
  transition: var(--transition);
}

.filters .chips {
  display: flex;
  gap: 0.5rem;
  flex-wrap: wrap;
}

.chip {
  border-radius: 999px;
  border: 1px solid var(--surface-border);
  background: transparent;
  color: var(--text-light);
  padding: 0.45rem 0.9rem;
  cursor: pointer;
  transition: var(--transition);
}

.chip.active {
  background: var(--button-ghost-bg);
  color: var(--text-primary);
  border-color: var(--button-ghost-border);
}

.filters .inline {
  display: flex;
  justify-content: space-between;
  flex-wrap: wrap;
  gap: 1rem;
  align-items: center;
}

.modal {
  position: fixed;
  inset: 0;
  background: var(--overlay-bg);
  display: flex;
  justify-content: center;
  align-items: center;
  padding: 2rem;
  z-index: 20;
}

.modal-content {
  background: var(--modal-bg);
  border-radius: 28px;
  border: 1px solid var(--card-border);
  width: min(700px, 100%);
  padding: 2rem;
  display: grid;
  gap: 1.5rem;
  box-shadow: 0 40px 90px var(--overlay-bg);
  transition: var(--transition);
}

.modal-content header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.modal-content h3 {
  font-size: 1.4rem;
  color: var(--text-primary);
}

.modal-content .icon {
  background: transparent;
  border: none;
  color: var(--text-muted);
  font-size: 1.5rem;
  cursor: pointer;
}

.modal-form {
  display: grid;
  gap: 1.25rem;
}

.modal-form label {
  display: grid;
  gap: 0.5rem;
  color: var(--text-light);
}

.modal-form input,
.modal-form textarea,
.modal-form select {
  border-radius: 14px;
  border: 1px solid var(--input-border);
  background: var(--input-bg);
  color: var(--input-text);
  padding: 0.75rem 1rem;
  transition: var(--transition);
}

.modal-form textarea {
  resize: vertical;
}

.modal-form .grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(220px, 1fr));
  gap: 1rem;
}

.modal-form footer {
  display: flex;
  justify-content: flex-end;
  gap: 1rem;
}

.empty-state {
  text-align: center;
  padding: 3rem;
  border-radius: 20px;
  border: 1px dashed var(--surface-border);
  color: var(--text-muted);
}

.defects-list {
  display: grid;
  gap: 1.5rem;
}

.defect-card {
  background: var(--card-bg);
  padding: 1.8rem;
  border-radius: 24px;
  border: 1px solid var(--card-border);
  display: grid;
  gap: 1.3rem;
  position: relative;
  overflow: hidden;
  transition: var(--transition);
}

.defect-card::before {
  content: '';
  position: absolute;
  inset: 0;
  border-radius: inherit;
  border: 1px solid rgba(59, 130, 246, 0.08);
  pointer-events: none;
}

.defect-card header {
  display: flex;
  justify-content: space-between;
  gap: 1rem;
}

.defect-card h3 {
  font-size: 1.3rem;
  color: var(--text-primary);
}

.defect-card .meta {
  color: var(--text-muted);
  font-size: 0.85rem;
  margin-top: 0.35rem;
}

.labels {
  display: flex;
  gap: 0.4rem;
  align-items: flex-start;
  flex-wrap: wrap;
}

.badge {
  border-radius: 999px;
  padding: 0.2rem 0.7rem;
  font-size: 0.75rem;
  letter-spacing: 0.3px;
  text-transform: uppercase;
}

.badge.priority.high { background: rgba(248, 113, 113, 0.18); color: var(--accent-danger); }
.badge.priority.medium { background: rgba(251, 191, 36, 0.18); color: var(--accent-warning); }
.badge.priority.low { background: rgba(52, 211, 153, 0.18); color: var(--accent-success); }

.badge.status.new { background: rgba(125, 211, 252, 0.16); color: var(--status-new); }
.badge.status.in_progress { background: rgba(96, 165, 250, 0.16); color: var(--status-in-progress); }
.badge.status.resolved { background: rgba(52, 211, 153, 0.16); color: var(--status-resolved); }
.badge.status.closed { background: rgba(233, 213, 255, 0.16); color: var(--status-closed); }

.badge.impact.low { background: rgba(59, 130, 246, 0.12); color: #bfdbfe; }
.badge.impact.medium { background: rgba(251, 191, 36, 0.12); color: #facc15; }
.badge.impact.critical { background: rgba(248, 113, 113, 0.18); color: #f87171; }

.description {
  color: rgba(226, 232, 240, 0.85);
  line-height: 1.6;
}

.tags {
  display: flex;
  flex-wrap: wrap;
  gap: 0.4rem;
  list-style: none;
}

.tags li {
  background: rgba(59, 130, 246, 0.16);
  color: #bfdbfe;
  border-radius: 999px;
  padding: 0.2rem 0.6rem;
  font-size: 0.75rem;
}

.details {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(160px, 1fr));
  gap: 1rem;
}

.details dt {
  font-size: 0.75rem;
  text-transform: uppercase;
  letter-spacing: 0.3px;
  color: rgba(148, 163, 184, 0.7);
}

.details dd {
  color: rgba(226, 232, 240, 0.9);
  margin-top: 0.25rem;
}

.history {
  border-top: 1px solid rgba(148, 163, 184, 0.12);
  padding-top: 1rem;
}

.history h4 {
  font-size: 0.85rem;
  text-transform: uppercase;
  letter-spacing: 0.3px;
  margin-bottom: 0.6rem;
  color: rgba(203, 213, 225, 0.75);
}

.history ul {
  list-style: none;
  display: grid;
  gap: 0.4rem;
  font-size: 0.85rem;
  color: rgba(148, 163, 184, 0.78);
}

.history li {
  display: flex;
  justify-content: space-between;
  gap: 1rem;
}

.actions {
  display: flex;
  flex-wrap: wrap;
  gap: 0.6rem;
}

.actions .ghost {
  padding: 0.5rem 0.9rem;
  font-size: 0.85rem;
}

.kanban {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
  gap: 1.2rem;
}

.kanban-column {
  background: rgba(12, 19, 33, 0.75);
  border-radius: 20px;
  border: 1px solid rgba(96, 165, 250, 0.18);
  display: grid;
  grid-template-rows: auto 1fr;
  max-height: 70vh;
}

.kanban-column header {
  display: flex;
  justify-content: space-between;
  padding: 1rem 1.2rem;
  border-bottom: 1px solid rgba(96, 165, 250, 0.12);
  color: #e2e8f0;
}

.kanban-cards {
  padding: 1.2rem;
  display: grid;
  gap: 0.8rem;
  overflow-y: auto;
}

.kanban-card {
  background: rgba(15, 23, 42, 0.68);
  border-radius: 16px;
  border: 1px solid rgba(59, 130, 246, 0.16);
  padding: 1rem;
  display: grid;
  gap: 0.5rem;
}

.kanban-card .title {
  color: #f8fafc;
  font-weight: 600;
}

.kanban-card .meta,
.kanban-card .assignee {
  font-size: 0.85rem;
  color: rgba(148, 163, 184, 0.75);
}

.kanban-card .mini {
  border: none;
  border-radius: 10px;
  padding: 0.35rem 0.6rem;
  background: rgba(96, 165, 250, 0.18);
  color: #bfdbfe;
  font-size: 0.75rem;
  cursor: pointer;
}

.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.2s ease;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}

@media (max-width: 480px) {
  .panel-header {
    flex-direction: column;
    align-items: flex-start;
    padding: 0.7rem 0.3rem;
  }
  .header-actions {
    gap: 0.4rem;
    padding: 0.2rem 0;
  }
  .quick-stats {
    grid-template-columns: 1fr;
    gap: 0.5rem;
  }
  .filters {
    padding: 0.5rem 0.2rem;
  }
  .field input, .field select {
    font-size: 0.95rem;
    padding: 0.5rem 0.7rem;
  }
  .stat {
    font-size: 0.95rem;
    padding: 0.5rem 0.2rem;
  }

  .panel-header {
    flex-direction: column;
    align-items: flex-start;
  }

  .header-actions {
    width: 100%;
    justify-content: flex-start;
    flex-wrap: wrap;
  }

  .quick-stats {
    grid-template-columns: 1fr;
  }

  .filters .inline {
    flex-direction: column;
    align-items: flex-start;
  }
}
</style>
