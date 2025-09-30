<template>
  <div class="projects-view">
    <header class="panel-header">
      <div>
        <h1>Объекты и команды</h1>
        <p>Создавайте объекты, следите за состоянием дефектов и планируйте контрольные точки.</p>
      </div>
      <button class="primary" @click="showCreate = true">Добавить объект</button>
    </header>

    <section class="portfolio">
      <article
        v-for="project in projectCards"
        :key="project.id"
        :class="['project-card', { active: selectedProject && selectedProject.id === project.id }]"
        @click="selectProject(project.id)"
      >
        <header>
          <div>
            <h3>{{ project.name }}</h3>
            <p>{{ project.address || 'Адрес не указан' }}</p>
          </div>
          <span :class="['badge', project.healthLevel]">{{ project.healthLabel }}</span>
        </header>
        <dl>
          <div>
            <dt>Открыто</dt>
            <dd>{{ project.openDefects }}</dd>
          </div>
          <div>
            <dt>Решено</dt>
            <dd>{{ project.resolvedDefects }}</dd>
          </div>
          <div>
            <dt>Просрочно</dt>
            <dd>{{ project.overdueDefects }}</dd>
          </div>
        </dl>
        <div class="progress">
          <div class="bar" :style="{ width: project.progress + '%' }"></div>
        </div>
        <footer>
          <span>{{ project.progress }}% выполнено</span>
          <span>{{ project.nextDeadline }}</span>
        </footer>
      </article>
    </section>

    <section v-if="selectedProject" class="project-details">
      <div class="card info">
        <header>
          <h2>{{ selectedProject.name }}</h2>
          <button type="button" class="ghost" @click="unselectProject">Закрыть</button>
        </header>
        <div class="grid">
          <div>
            <span class="label">Адрес</span>
            <p>{{ selectedProject.address || 'Не указан' }}</p>
          </div>
          <div>
            <span class="label">Ответственный</span>
            <p>{{ selectedProject.supervisor || 'Назначьте ответственного' }}</p>
          </div>
          <div>
            <span class="label">Начало</span>
            <p>{{ formatDate(selectedProject.startDate) }}</p>
          </div>
          <div>
            <span class="label">Сдача</span>
            <p>{{ formatDate(selectedProject.dueDate) }}</p>
          </div>
        </div>
        <p class="description">{{ selectedProject.description || 'Добавьте заметки о проекте, чтобы команда была в курсе контекста.' }}</p>
      </div>

      <div class="card defects">
        <header>
          <h3>Сводка по дефектам</h3>
        </header>
        <ul class="defects-list">
          <li v-for="item in projectTimeline" :key="item.id">
            <div>
              <p class="title">{{ item.title }}</p>
              <p class="meta">{{ item.statusLabel }} · {{ item.priorityLabel }}</p>
            </div>
            <span class="deadline">{{ item.deadlineLabel }}</span>
          </li>
          <li v-if="!projectTimeline.length" class="empty">Дефектов не найдено</li>
        </ul>
      </div>

      <div class="card actions" v-if="canManage">
        <h3>Действия</h3>
        <div class="buttons">
          <button class="ghost" @click="duplicateProject(selectedProject.id)">Дублировать объект</button>
          <button class="ghost danger" @click="removeProject(selectedProject.id)">Удалить объект</button>
        </div>
      </div>
    </section>

    <transition name="fade">
      <div v-if="showCreate" class="modal">
        <div class="modal-content">
          <header>
            <h3>Новый объект</h3>
            <button type="button" class="icon" @click="closeCreate">×</button>
          </header>
          <form @submit.prevent="createProject" class="modal-form">
            <label>
              <span>Название</span>
              <input v-model="newProject.name" required placeholder="Например, ЖК «Северный меридиан»">
            </label>
            <label>
              <span>Адрес</span>
              <input v-model="newProject.address" placeholder="Город, улица, дом">
            </label>
            <div class="grid">
              <label>
                <span>Ответственный</span>
                <input v-model="newProject.supervisor" placeholder="ФИО или подразделение">
              </label>
              <label>
                <span>Дата начала</span>
                <input v-model="newProject.startDate" type="date">
              </label>
              <label>
                <span>Срок сдачи</span>
                <input v-model="newProject.dueDate" type="date">
              </label>
            </div>
            <label>
              <span>Описание</span>
              <textarea v-model="newProject.description" rows="3" placeholder="Цели объекта, критические зоны, особенности логистики"></textarea>
            </label>
            <footer>
              <button type="button" class="ghost" @click="closeCreate">Отмена</button>
              <button type="submit" class="primary">Создать</button>
            </footer>
          </form>
        </div>
      </div>
    </transition>
  </div>
</template>

<script>
import { computed, onBeforeUnmount, onMounted, reactive, ref, watch } from 'vue'

export default {
  name: 'ProjectsView',
  emits: ['sync'],
  props: {
    user: {
      type: Object,
      required: true
    }
  },
  setup(props, { emit }) {
    const projects = ref([])
    const defects = ref([])
    const showCreate = ref(false)
    const selectedProjectId = ref(null)

    const newProject = reactive({
      name: '',
      address: '',
      supervisor: '',
      startDate: '',
      dueDate: '',
      description: ''
    })

    const canManage = computed(() => ['manager', 'admin'].includes(props.user.role))

    const normalizeProject = (project) => ({
      supervisor: '',
      startDate: '',
      dueDate: '',
      description: '',
      createdAt: new Date().toISOString(),
      ...project
    })

    const refreshFromStorage = () => {
      const storedProjects = JSON.parse(localStorage.getItem('projects') || '[]')
      const storedDefects = JSON.parse(localStorage.getItem('defects') || '[]')

      projects.value = storedProjects.map(normalizeProject)
      defects.value = storedDefects
    }

    const saveProjects = () => {
      localStorage.setItem('projects', JSON.stringify(projects.value))
      emit('sync')
    }

    const saveDefects = () => {
      localStorage.setItem('defects', JSON.stringify(defects.value))
      emit('sync')
    }

    const projectCards = computed(() => {
      return projects.value.map(project => {
        const projectDefects = defects.value.filter(defect => defect.projectId === project.id)
        const openDefects = projectDefects.filter(defect => ['new', 'in_progress'].includes(defect.status))
        const resolved = projectDefects.filter(defect => ['resolved', 'closed'].includes(defect.status))
        const overdue = projectDefects.filter(defect => defect.deadline && new Date(defect.deadline) < new Date() && !['resolved', 'closed'].includes(defect.status))

        const progress = projectDefects.length
          ? Math.round((resolved.length / projectDefects.length) * 100)
          : 0

        const nextDeadline = () => {
          if (!projectDefects.length) return 'Нет задач'
          const future = projectDefects
            .filter(defect => defect.deadline)
            .sort((a, b) => new Date(a.deadline) - new Date(b.deadline))
          if (!future.length) return 'Дедлайны не назначены'
          return new Intl.DateTimeFormat('ru-RU', { day: '2-digit', month: '2-digit' }).format(new Date(future[0].deadline))
        }

        const healthLevel = () => {
          if (overdue.length > 0) return 'alert'
          if (progress >= 70) return 'good'
          if (progress >= 40) return 'warn'
          return 'neutral'
        }

        const healthLabelMap = {
          alert: 'Требует внимания',
          warn: 'Под контролем',
          good: 'Уверенный прогресс',
          neutral: 'Старт работ'
        }

        const level = healthLevel()

        return {
          ...project,
          progress,
          openDefects: openDefects.length,
          resolvedDefects: resolved.length,
          overdueDefects: overdue.length,
          nextDeadline: nextDeadline(),
          healthLevel: level,
          healthLabel: healthLabelMap[level]
        }
      })
    })

    const selectedProject = computed(() => {
      if (!selectedProjectId.value) return null
      return projectCards.value.find(project => project.id === selectedProjectId.value) || null
    })

    const projectTimeline = computed(() => {
      if (!selectedProject.value) return []
      const projectDefects = defects.value.filter(defect => defect.projectId === selectedProject.value.id)
      return projectDefects
        .map(defect => ({
          id: defect.id,
          title: defect.title,
          statusLabel: statusLabels[defect.status] || defect.status,
          priorityLabel: priorityLabels[defect.priority] || defect.priority,
          deadlineLabel: formatDeadline(defect.deadline)
        }))
        .sort((a, b) => a.deadlineLabel.localeCompare(b.deadlineLabel))
    })

    const statusLabels = {
      new: 'Новый',
      in_progress: 'В работе',
      resolved: 'Решен',
      closed: 'Закрыт'
    }

    const priorityLabels = {
      low: 'Низкий',
      medium: 'Средний',
      high: 'Высокий'
    }

    const selectProject = (projectId) => {
      selectedProjectId.value = projectId
    }

    const unselectProject = () => {
      selectedProjectId.value = null
    }

    const createProject = () => {
      const id = Date.now()
      projects.value.unshift(normalizeProject({ id, ...newProject }))
      closeCreate()
      selectedProjectId.value = id
    }

    const duplicateProject = (projectId) => {
      const project = projects.value.find(item => item.id === projectId)
      if (!project) return
      const id = Date.now()
      projects.value.unshift({ ...project, id, name: `${project.name} (копия)` })
      selectedProjectId.value = id
    }

    const removeProject = (projectId) => {
      projects.value = projects.value.filter(project => project.id !== projectId)
      defects.value = defects.value.filter(defect => defect.projectId !== projectId)
      if (selectedProjectId.value === projectId) {
        selectedProjectId.value = null
      }
      saveDefects()
    }

    const closeCreate = () => {
      showCreate.value = false
      Object.assign(newProject, {
        name: '',
        address: '',
        supervisor: '',
        startDate: '',
        dueDate: '',
        description: ''
      })
    }

    const formatDate = (value) => {
      if (!value) return 'Не задано'
      return new Intl.DateTimeFormat('ru-RU').format(new Date(value))
    }

    const formatDeadline = (value) => {
      if (!value) return 'Без срока'
      return new Intl.DateTimeFormat('ru-RU').format(new Date(value))
    }

    watch(projects, saveProjects, { deep: true })
    watch(defects, saveDefects, { deep: true })

    onMounted(() => {
      refreshFromStorage()
      window.addEventListener('storage', refreshFromStorage)
    })

    onBeforeUnmount(() => {
      window.removeEventListener('storage', refreshFromStorage)
    })

    return {
      projectCards,
      selectedProject,
      projectTimeline,
      formatDate,
      formatDeadline,
      selectProject,
      unselectProject,
      showCreate,
      newProject,
      createProject,
      closeCreate,
      duplicateProject,
      removeProject,
      canManage,
      refreshFromStorage
    }
  }
}
</script>

<style scoped>
.panel-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  margin-bottom: 2rem;
  gap: 1.5rem;
}

.panel-header h1 {
  font-size: 1.8rem;
  color: #f8fafc;
}

.panel-header p {
  color: rgba(148, 163, 184, 0.75);
  margin-top: 0.35rem;
}

.primary {
  border: none;
  border-radius: 16px;
  padding: 0.85rem 1.4rem;
  background: linear-gradient(135deg, #34d399 0%, #60a5fa 100%);
  color: #041224;
  font-weight: 700;
  cursor: pointer;
  transition: transform 0.2s ease;
}

.primary:hover {
  transform: translateY(-1px);
}

.portfolio {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(260px, 1fr));
  gap: 1.5rem;
}

.project-card {
  background: linear-gradient(180deg, rgba(17,24,39,0.6) 0%, rgba(17,24,39,0.35) 100%);
  border: 1px solid rgba(96, 165, 250, 0.18);
  border-radius: 24px;
  padding: 1.6rem;
  display: grid;
  gap: 1.1rem;
  cursor: pointer;
  transition: transform 0.2s ease, border 0.2s ease, box-shadow 0.2s ease;
}

.project-card.active,
.project-card:hover {
  transform: translateY(-4px);
  border-color: rgba(96, 165, 250, 0.35);
  box-shadow: 0 14px 28px rgba(15, 23, 42, 0.45);
}

.project-card header {
  display: flex;
  justify-content: space-between;
  gap: 1rem;
}

.project-card h3 {
  font-size: 1.2rem;
  color: #f8fafc;
}

.project-card p {
  color: rgba(148, 163, 184, 0.75);
  font-size: 0.9rem;
}

.badge {
  height: fit-content;
  border-radius: 999px;
  padding: 0.3rem 0.75rem;
  font-size: 0.75rem;
  text-transform: uppercase;
  letter-spacing: 0.4px;
}

.badge.good { background: rgba(74, 222, 128, 0.18); color: #86efac; }
.badge.warn { background: rgba(250, 204, 21, 0.18); color: #facc15; }
.badge.alert { background: rgba(248, 113, 113, 0.2); color: #fca5a5; }
.badge.neutral { background: rgba(148, 163, 184, 0.18); color: #cbd5f5; }

.project-card dl {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 0.8rem;
}

.project-card dt {
  font-size: 0.75rem;
  text-transform: uppercase;
  letter-spacing: 0.3px;
  color: rgba(148, 163, 184, 0.7);
}

.project-card dd {
  color: #f8fafc;
  font-weight: 600;
}

.progress {
  height: 8px;
  border-radius: 999px;
  background: rgba(148, 163, 184, 0.18);
  overflow: hidden;
}

.progress .bar {
  height: 100%;
  background: linear-gradient(90deg, #38bdf8 0%, #34d399 100%);
}

.project-card footer {
  display: flex;
  justify-content: space-between;
  font-size: 0.85rem;
  color: rgba(148, 163, 184, 0.75);
}

.project-details {
  margin-top: 2.5rem;
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
  gap: 1.5rem;
}

.card {
  background: rgba(12, 19, 33, 0.7);
  border-radius: 24px;
  border: 1px solid rgba(96, 165, 250, 0.16);
  padding: 1.8rem;
  display: grid;
  gap: 1.2rem;
}

.card header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
}

.card h2 {
  color: #f8fafc;
  font-size: 1.4rem;
}

.card .grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(180px, 1fr));
  gap: 1rem;
}

.card .label {
  font-size: 0.75rem;
  text-transform: uppercase;
  color: rgba(148, 163, 184, 0.66);
  letter-spacing: 0.3px;
}

.card .description {
  color: rgba(203, 213, 225, 0.78);
  line-height: 1.6;
}

.defects-list {
  list-style: none;
  display: grid;
  gap: 0.9rem;
}

.defects-list li {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 1.2rem;
  background: rgba(15, 23, 42, 0.55);
  border-radius: 16px;
  padding: 0.9rem 1.2rem;
  border: 1px solid rgba(96, 165, 250, 0.12);
}

.defects-list .title {
  color: #f8fafc;
  font-weight: 600;
}

.defects-list .meta {
  color: rgba(148, 163, 184, 0.75);
  font-size: 0.85rem;
}

.defects-list .deadline {
  font-size: 0.8rem;
  color: rgba(203, 213, 225, 0.75);
}

.defects-list .empty {
  justify-content: center;
  color: rgba(148, 163, 184, 0.75);
}

.ghost {
  border: 1px solid rgba(148, 163, 184, 0.3);
  background: transparent;
  color: rgba(203, 213, 225, 0.9);
  border-radius: 14px;
  padding: 0.6rem 1rem;
  cursor: pointer;
}

.ghost.danger {
  border-color: rgba(248, 113, 113, 0.35);
  color: #fca5a5;
}

.buttons {
  display: flex;
  flex-direction: column;
  gap: 0.6rem;
}

.modal {
  position: fixed;
  inset: 0;
  background: rgba(6, 11, 20, 0.78);
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 2rem;
  z-index: 30;
}

.modal-content {
  width: min(640px, 100%);
  background: linear-gradient(180deg, rgba(15, 23, 42, 0.95) 0%, rgba(15, 23, 42, 0.75) 100%);
  border-radius: 28px;
  border: 1px solid rgba(96, 165, 250, 0.25);
  padding: 2rem;
  display: grid;
  gap: 1.5rem;
  box-shadow: 0 40px 90px rgba(2, 6, 23, 0.45);
}

.modal-content header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.modal-content h3 {
  font-size: 1.4rem;
  color: #f8fafc;
}

.modal-content .icon {
  background: transparent;
  border: none;
  color: rgba(148, 163, 184, 0.7);
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
  color: rgba(203, 213, 225, 0.82);
}

.modal-form input,
.modal-form textarea {
  border-radius: 14px;
  border: 1px solid rgba(148, 163, 184, 0.25);
  background: rgba(15, 23, 42, 0.65);
  color: #f8fafc;
  padding: 0.75rem 1rem;
}

.modal-form textarea {
  resize: vertical;
}

.modal-form .grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(180px, 1fr));
  gap: 1rem;
}

.modal-form footer {
  display: flex;
  justify-content: flex-end;
  gap: 1rem;
}

.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.25s ease;
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
  .portfolio {
    grid-template-columns: 1fr;
    gap: 0.5rem;
    padding: 0.2rem;
  }
  .project-card {
    padding: 0.7rem 0.4rem;
    font-size: 0.97rem;
  }
  .project-details {
    grid-template-columns: 1fr;
    padding: 0.5rem 0.2rem;
  }
  .card {
    padding: 0.7rem 0.4rem;
  }

</style>
