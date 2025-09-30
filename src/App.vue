<template>
  <div class="app-root">
    <div v-if="!user" class="landing">
      <section class="landing-intro">
        <h1>Pulse QA Control</h1>
        <p>
          Панель управления качеством строительства: визуальный контроль дефектов,
          управление объектами и аналитика без сервера и сложной настройки.
        </p>
        <ul>
          <li>Ведите учет дефектов с дедлайнами и ответственными</li>
          <li>Отслеживайте динамику выполнения работ</li>
          <li>Используйте офлайн-хранилище: данные остаются на вашем устройстве</li>
        </ul>
      </section>
      <LoginView @login="handleLogin" />
    </div>

    <div v-else class="app-shell">
      <aside class="app-sidebar">
        <div class="brand">
          <span class="brand-icon">⛏️</span>
          <div>
            <strong>Pulse QA</strong>
          </div>
        </div>

        <div class="sidebar-user">
          <div class="user-badge">
            <span class="avatar">{{ initials }}</span>
            <div>
              <p class="user-name">{{ user.name }}</p>
              <p class="user-role">{{ getRoleTitle(user.role) }}</p>
            </div>
          </div>
        </div>

        <nav class="sidebar-nav">
          <button
            v-for="item in navItems"
            :key="item.id"
            :class="['nav-item', { active: currentView === item.id }]"
            @click="currentView = item.id"
          >
            <span class="nav-icon">{{ item.icon }}</span>
            <span>{{ item.label }}</span>
            <span v-if="item.id === 'defects' && metrics.totalDefects" class="nav-counter">{{ metrics.totalDefects }}</span>
          </button>
        </nav>

        <button class="logout" @click="logout">Выйти</button>
      </aside>

      <div class="app-content">
        <header class="top-bar">
          <div>
            <h2>{{ currentTitle }}</h2>
            <p>Последняя синхронизация: {{ lastSyncLabel }}</p>
          </div>
          <div class="top-actions">
            <button @click="refreshMetrics" class="ghost">Обновить данные</button>
          </div>
        </header>

        <main class="view-area">
          <section v-if="currentView === 'overview'" class="overview">
            <div class="pulse-grid">
              <div class="pulse-card highlight">
                <h3>Общий прогресс</h3>
                <p class="headline">{{ metrics.totalDefects }} активных дефектов</p>
                <div class="progress">
                  <div class="progress-bar" :style="{ width: metrics.recoveryRate + '%' }"></div>
                </div>
                <p class="caption">{{ metrics.recoveryRate }}% решений (решено + закрыто)</p>
              </div>

              <div class="pulse-card" v-for="(value, key) in metrics.statuses" :key="key">
                <h3>{{ statusLabels[key] }}</h3>
                <p class="metric">{{ value }}</p>
              </div>
            </div>

            <div class="overview-columns">
              <div class="pulse-card">
                <header>
                  <h3>Ближайшие дедлайны</h3>
                  <span class="badge">{{ metrics.upcomingDeadlines.length }}</span>
                </header>
                <ul v-if="metrics.upcomingDeadlines.length" class="mini-list">
                  <li v-for="deadline in metrics.upcomingDeadlines" :key="deadline.id">
                    <div>
                      <p class="item-title">{{ deadline.title }}</p>
                      <p class="item-meta">{{ deadline.projectName }} · {{ deadline.assignee || 'Не назначен' }}</p>
                    </div>
                    <span :class="['pill', deadline.daysLeft < 0 ? 'danger' : deadline.daysLeft <= 2 ? 'warning' : 'good']">
                      {{ deadline.label }}
                    </span>
                  </li>
                </ul>
                <p v-else class="empty">Нет назначенных дедлайнов</p>
              </div>

              <div class="pulse-card">
                <header>
                  <h3>Сильные проекты</h3>
                  <span class="badge">ТОП {{ metrics.projectBreakdown.length }}</span>
                </header>
                <ul v-if="metrics.projectBreakdown.length" class="mini-list">
                  <li v-for="project in metrics.projectBreakdown" :key="project.id">
                    <div>
                      <p class="item-title">{{ project.name }}</p>
                      <p class="item-meta">{{ project.address || 'Адрес не указан' }}</p>
                    </div>
                    <div class="progress-line">
                      <span>{{ project.resolved }}/{{ project.total }}</span>
                      <div class="line">
                        <span class="fill" :style="{ width: project.progress + '%' }"></span>
                      </div>
                    </div>
                  </li>
                </ul>
                <p v-else class="empty">Добавьте первый проект, чтобы увидеть статистику</p>
              </div>

              <div class="pulse-card">
                <header>
                  <h3>Последние изменения</h3>
                </header>
                <ul v-if="metrics.recentDefects.length" class="timeline">
                  <li v-for="item in metrics.recentDefects" :key="item.id">
                    <span class="dot"></span>
                    <div>
                      <p class="item-title">{{ item.title }}</p>
                      <p class="item-meta">{{ item.projectName }} · {{ item.statusLabel }}</p>
                      <small>{{ item.when }}</small>
                    </div>
                  </li>
                </ul>
                <p v-else class="empty">Создайте дефект, чтобы начать журнал активности</p>
              </div>
            </div>
          </section>

          <DefectsView
            v-else-if="currentView === 'defects'"
            :user="user"
            @sync="refreshMetrics"
          />

          <ProjectsView
            v-else-if="currentView === 'projects'"
            :user="user"
            @sync="refreshMetrics"
          />

          <ReportsView
            v-else-if="currentView === 'reports'"
            :user="user"
            @sync="refreshMetrics"
          />
        </main>
      </div>
    </div>
  </div>
</template>

<script>
import { computed, onBeforeUnmount, onMounted, reactive, ref } from 'vue'
import LoginView from './components/LoginView.vue'
import DefectsView from './components/DefectsView.vue'
import ProjectsView from './components/ProjectsView.vue'
import ReportsView from './components/ReportsView.vue'

export default {
  name: 'App',
  components: {
    LoginView,
    DefectsView,
    ProjectsView,
    ReportsView
  },
  setup() {
    const user = ref(null)
    const currentView = ref('overview')
    const lastSync = ref(null)

    const statusLabels = {
      new: 'Новые заявки',
      in_progress: 'В работе',
      resolved: 'Решены',
      closed: 'Закрыты'
    }

    const metrics = reactive({
      totalDefects: 0,
      statuses: {
        new: 0,
        in_progress: 0,
        resolved: 0,
        closed: 0
      },
      recoveryRate: 0,
      projectBreakdown: [],
      upcomingDeadlines: [],
      recentDefects: []
    })

    const navItems = [
      { id: 'overview', label: 'Панель', icon: '📊' },
      { id: 'defects', label: 'Дефекты', icon: '🧪' },
      { id: 'projects', label: 'Объекты', icon: '🏗️' },
      { id: 'reports', label: 'Отчеты', icon: '📈' }
    ]

    const getRoleTitle = (role) => {
      const roles = {
        engineer: 'Инженер контроля',
        manager: 'Руководитель проекта',
        admin: 'Руководитель направления'
      }
      return roles[role] || 'Специалист'
    }

    const initials = computed(() => {
      if (!user.value?.name) return '??'
      return user.value.name
        .split(' ')
        .map(part => part.charAt(0).toUpperCase())
        .slice(0, 2)
        .join('')
    })

    const currentTitle = computed(() => {
      const titles = {
        overview: 'Обзор качества',
        defects: 'Центр управления дефектами',
        projects: 'Объекты и команды',
        reports: 'Аналитика и отчеты'
      }
      return titles[currentView.value] || 'Панель управления'
    })

    const lastSyncLabel = computed(() => {
      if (!lastSync.value) return 'данные не обновлялись'
      return new Intl.DateTimeFormat('ru-RU', {
        hour: '2-digit',
        minute: '2-digit'
      }).format(lastSync.value)
    })

    const normalizeDefect = (defect) => ({
      tags: [],
      assignee: '',
      deadline: '',
      status: 'new',
      priority: 'medium',
      createdAt: new Date().toISOString(),
      ...defect
    })

    const refreshMetrics = () => {
      const storedDefects = JSON.parse(localStorage.getItem('defects') || '[]')
      const storedProjects = JSON.parse(localStorage.getItem('projects') || '[]')

      const normalizedDefects = storedDefects.map(normalizeDefect)
      if (storedDefects.length !== normalizedDefects.length || JSON.stringify(storedDefects) !== JSON.stringify(normalizedDefects)) {
        localStorage.setItem('defects', JSON.stringify(normalizedDefects))
      }

      const defectsByStatus = normalizedDefects.reduce(
        (acc, defect) => {
          acc[defect.status] = (acc[defect.status] || 0) + 1
          return acc
        },
        { new: 0, in_progress: 0, resolved: 0, closed: 0 }
      )

      metrics.totalDefects = normalizedDefects.length
      metrics.statuses = { ...metrics.statuses, ...defectsByStatus }

      const completed = metrics.statuses.resolved + metrics.statuses.closed
      metrics.recoveryRate = metrics.totalDefects === 0
        ? 0
        : Math.round((completed / metrics.totalDefects) * 100)

      const projectMap = new Map(storedProjects.map(project => [project.id, project]))
      metrics.projectBreakdown = storedProjects
        .map(project => {
          const projectDefects = normalizedDefects.filter(defect => defect.projectId === project.id)
          const resolved = projectDefects.filter(defect => ['resolved', 'closed'].includes(defect.status)).length
          return {
            id: project.id,
            name: project.name,
            address: project.address,
            total: projectDefects.length,
            resolved,
            progress: projectDefects.length ? Math.round((resolved / projectDefects.length) * 100) : 0
          }
        })
        .sort((a, b) => b.total - a.total)
        .slice(0, 4)

      const upcoming = normalizedDefects
        .filter(defect => defect.deadline)
        .map(defect => {
          const project = projectMap.get(defect.projectId)
          const daysLeft = Math.ceil(
            (new Date(defect.deadline).getTime() - new Date().setHours(0, 0, 0, 0)) /
            (1000 * 60 * 60 * 24)
          )
          const label = daysLeft < 0
            ? `Просрочен на ${Math.abs(daysLeft)} д.`
            : daysLeft === 0
              ? 'Сегодня'
              : `Через ${daysLeft} д.`
          return {
            id: defect.id,
            title: defect.title,
            projectName: project?.name || 'Проект не указан',
            assignee: defect.assignee,
            daysLeft,
            label
          }
        })
        .sort((a, b) => a.daysLeft - b.daysLeft)
        .slice(0, 4)

      metrics.upcomingDeadlines = upcoming

      metrics.recentDefects = normalizedDefects
        .slice()
        .sort((a, b) => new Date(b.createdAt).getTime() - new Date(a.createdAt).getTime())
        .slice(0, 6)
        .map(defect => {
          const project = projectMap.get(defect.projectId)
          return {
            id: defect.id,
            title: defect.title,
            projectName: project?.name || 'Без проекта',
            statusLabel: statusLabels[defect.status] || defect.status,
            when: new Intl.DateTimeFormat('ru-RU', {
              day: '2-digit', month: '2-digit', hour: '2-digit', minute: '2-digit'
            }).format(new Date(defect.createdAt))
          }
        })

      lastSync.value = new Date()
    }

    const handleLogin = (userData) => {
      user.value = userData
      currentView.value = 'overview'
      refreshMetrics()
    }

    const logout = () => {
      user.value = null
      currentView.value = 'overview'
    }

    const storageListener = (event) => {
      if (['defects', 'projects'].includes(event.key)) {
        refreshMetrics()
      }
    }

    onMounted(() => {
      refreshMetrics()
      window.addEventListener('storage', storageListener)
    })

    onBeforeUnmount(() => {
      window.removeEventListener('storage', storageListener)
    })

    return {
      user,
      currentView,
      navItems,
      metrics,
      statusLabels,
      getRoleTitle,
      handleLogin,
      logout,
      refreshMetrics,
      initials,
      currentTitle,
      lastSyncLabel
    }
  }
}
</script>

<style>
@import url('https://fonts.googleapis.com/css2?family=Manrope:wght@400;600;700&display=swap');

* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: 'Manrope', sans-serif;
  background: radial-gradient(120% 120% at 10% 10%, #1f2a40 0%, #0f1729 55%, #05070c 100%);
  color: #f6f7fb;
  min-height: 100vh;
}

.app-root {
  min-height: 100vh;
  display: flex;
  align-items: stretch;
  justify-content: center;
  padding: 2rem;
}

.landing {
  display: grid;
  grid-template-columns: minmax(300px, 420px) minmax(360px, 420px);
  gap: 3rem;
  width: 100%;
  max-width: 1000px;
  align-items: center;
}

.landing-intro {
  background: linear-gradient(160deg, rgba(255,255,255,0.06) 0%, rgba(255,255,255,0.02) 100%);
  border: 1px solid rgba(255,255,255,0.08);
  border-radius: 24px;
  padding: 2.5rem;
  backdrop-filter: blur(16px);
  box-shadow: 0 30px 60px rgba(3, 10, 24, 0.45);
}

.landing-intro h1 {
  font-size: 2.4rem;
  margin-bottom: 1rem;
  color: #f8fafc;
}

.landing-intro p {
  color: rgba(226, 232, 240, 0.86);
  line-height: 1.6;
  margin-bottom: 1.5rem;
}

.landing-intro ul {
  list-style: none;
  display: grid;
  gap: 0.75rem;
}

.landing-intro li {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  color: rgba(212, 223, 239, 0.9);
}

.landing-intro li::before {
  content: '•';
  color: #60a5fa;
  font-size: 1.4rem;
}

.app-shell {
  width: 100%;
  max-width: 1280px;
  display: grid;
  grid-template-columns: 280px 1fr;
  border-radius: 32px;
  border: 1px solid rgba(255,255,255,0.08);
  overflow: hidden;
  background: rgba(10, 16, 28, 0.82);
  box-shadow: 0 40px 70px rgba(0, 0, 0, 0.55);
  backdrop-filter: blur(22px);
}

.app-sidebar {
  background: linear-gradient(200deg, rgba(24, 33, 56, 0.95) 0%, rgba(12, 18, 31, 0.92) 100%);
  padding: 2.5rem 2rem;
  display: flex;
  flex-direction: column;
  gap: 2rem;
}

.brand {
  display: flex;
  align-items: center;
  gap: 1rem;
  font-size: 1.1rem;
  letter-spacing: 0.5px;
}

.brand-icon {
  font-size: 1.8rem;
}

.brand small {
  color: rgba(148, 163, 184, 0.9);
  font-size: 0.8rem;
}

.sidebar-user {
  display: grid;
  gap: 0.9rem;
}

.user-badge {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.avatar {
  width: 48px;
  height: 48px;
  border-radius: 16px;
  background: rgba(148,163,184,0.16);
  display: grid;
  place-items: center;
  font-weight: 700;
  color: #60a5fa;
}

.user-name {
  font-weight: 600;
  color: #e2e8f0;
}

.user-role {
  font-size: 0.85rem;
  color: rgba(148, 163, 184, 0.85);
}

.user-message {
  font-size: 0.9rem;
  color: rgba(209, 213, 219, 0.78);
}

.sidebar-nav {
  display: grid;
  gap: 0.5rem;
  flex: 1;
}

.nav-item {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  padding: 0.85rem 1rem;
  border-radius: 14px;
  border: none;
  background: transparent;
  color: rgba(226,232,240,0.85);
  font-size: 0.95rem;
  cursor: pointer;
  transition: all 0.2s ease;
}

.nav-item:hover,
.nav-item.active {
  background: rgba(96, 165, 250, 0.14);
  color: #f8fafc;
}

.nav-icon {
  font-size: 1.1rem;
}

.nav-counter {
  margin-left: auto;
  background: rgba(96, 165, 250, 0.12);
  border-radius: 999px;
  padding: 0.15rem 0.6rem;
  font-size: 0.75rem;
  color: #a5b4fc;
}

.logout {
  background: rgba(248, 250, 252, 0.08);
  color: rgba(255, 255, 255, 0.8);
  border: 1px solid rgba(255, 255, 255, 0.12);
  border-radius: 12px;
  padding: 0.75rem 1rem;
  cursor: pointer;
  transition: background 0.2s ease;
}

.logout:hover {
  background: rgba(248, 250, 252, 0.18);
}

.app-content {
  background: rgba(9, 13, 22, 0.7);
  display: grid;
  grid-template-rows: auto 1fr;
  backdrop-filter: blur(18px);
}

.top-bar {
  padding: 2.5rem 3rem 1.5rem;
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
}

.top-bar h2 {
  font-size: 1.8rem;
  font-weight: 700;
  color: #e2e8f0;
}

.top-bar p {
  color: rgba(148, 163, 184, 0.75);
  margin-top: 0.45rem;
  font-size: 0.9rem;
}

.top-actions {
  display: flex;
  gap: 0.75rem;
}

.ghost {
  border: 1px solid rgba(96, 165, 250, 0.25);
  background: rgba(96, 165, 250, 0.08);
  color: #bfdbfe;
  border-radius: 12px;
  padding: 0.65rem 1.1rem;
  cursor: pointer;
  transition: all 0.2s ease;
}

.ghost:hover {
  background: rgba(96, 165, 250, 0.16);
}

.view-area {
  padding: 0 3rem 3rem;
  overflow-y: auto;
}

.overview {
  display: grid;
  gap: 2rem;
}

.pulse-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(220px, 1fr));
  gap: 1.5rem;
}

.pulse-card {
  background: linear-gradient(180deg, rgba(17, 24, 39, 0.65) 0%, rgba(17, 24, 39, 0.35) 100%);
  border: 1px solid rgba(148, 163, 184, 0.1);
  border-radius: 24px;
  padding: 1.75rem;
  display: grid;
  gap: 1rem;
  box-shadow: inset 0 1px 0 rgba(148, 163, 184, 0.16);
}

.pulse-card header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.pulse-card.highlight {
  grid-column: span 2;
  background: linear-gradient(145deg, rgba(37, 99, 235, 0.35) 0%, rgba(14, 116, 144, 0.25) 100%);
}

.pulse-card h3 {
  color: #cbd5f5;
  font-size: 1rem;
  letter-spacing: 0.4px;
  text-transform: uppercase;
}

.headline {
  font-size: 1.6rem;
  font-weight: 700;
  color: #f8fafc;
}

.metric {
  font-size: 2.4rem;
  font-weight: 700;
  color: #fbbf24;
}

.progress {
  width: 100%;
  height: 10px;
  border-radius: 999px;
  background: rgba(15, 23, 42, 0.7);
  overflow: hidden;
}

.progress-bar {
  height: 100%;
  background: linear-gradient(90deg, #38bdf8 0%, #4ade80 100%);
  border-radius: inherit;
  transition: width 0.3s ease;
}

.caption {
  color: rgba(226, 232, 240, 0.65);
  font-size: 0.9rem;
}

.overview-columns {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(260px, 1fr));
  gap: 1.5rem;
}

.badge {
  background: rgba(96, 165, 250, 0.18);
  color: #bfdbfe;
  border-radius: 1rem;
  padding: 0.2rem 0.75rem;
  font-size: 0.75rem;
  letter-spacing: 0.3px;
}

.mini-list {
  list-style: none;
  display: grid;
  gap: 1.1rem;
}

.mini-list li {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 1.4rem;
}

.item-title {
  font-weight: 600;
  color: #e2e8f0;
}

.item-meta {
  font-size: 0.85rem;
  color: rgba(148, 163, 184, 0.78);
  margin-top: 0.3rem;
}

.progress-line {
  display: grid;
  gap: 0.5rem;
  width: 140px;
}

.progress-line .line {
  height: 6px;
  background: rgba(148, 163, 184, 0.18);
  border-radius: 999px;
  overflow: hidden;
}

.progress-line .fill {
  display: block;
  height: 100%;
  background: linear-gradient(90deg, #4ade80 0%, #38bdf8 100%);
}

.pill {
  border-radius: 999px;
  padding: 0.35rem 0.8rem;
  font-size: 0.8rem;
  font-weight: 600;
  letter-spacing: 0.2px;
}

.pill.danger { background: rgba(248, 113, 113, 0.18); color: #fca5a5; }
.pill.warning { background: rgba(251, 191, 36, 0.2); color: #facc15; }
.pill.good { background: rgba(52, 211, 153, 0.18); color: #34d399; }

.timeline {
  list-style: none;
  display: grid;
  gap: 1.2rem;
  position: relative;
  padding-left: 1.4rem;
}

.timeline::before {
  content: '';
  position: absolute;
  top: 0.2rem;
  left: 0.45rem;
  width: 1px;
  bottom: 0.2rem;
  background: rgba(148, 163, 184, 0.25);
}

.timeline li {
  position: relative;
  display: grid;
  gap: 0.3rem;
}

.timeline .dot {
  width: 10px;
  height: 10px;
  border-radius: 50%;
  background: linear-gradient(90deg, #38bdf8 0%, #4ade80 100%);
  position: absolute;
  left: -1.4rem;
  top: 0.3rem;
}

.timeline small {
  color: rgba(148, 163, 184, 0.65);
  font-size: 0.75rem;
}

.empty {
  color: rgba(148, 163, 184, 0.6);
  font-size: 0.9rem;
}

@media (max-width: 1100px) {
  .app-shell {
    grid-template-columns: 1fr;
  }

  .app-sidebar {
    flex-direction: row;
    overflow-x: auto;
    align-items: center;
    gap: 1.5rem;
  }

  .sidebar-nav {
    grid-auto-flow: column;
    grid-auto-columns: 1fr;
  }

  .logout {
    flex-shrink: 0;
  }

  .app-content {
    grid-row: 2 / span 1;
  }
}

@media (max-width: 860px) {
  .landing {
    grid-template-columns: 1fr;
  }

  .app-root {
    padding: 1.5rem;
  }
}

</style>

/* --- Мобильная адаптация --- */
@media (max-width: 480px) {
  .app-root {
    padding: 0.5rem;
  }
  .app-shell {
    grid-template-columns: 1fr;
  }
  .app-sidebar {
    min-width: 0;
    padding: 0.5rem 0.2rem;
    gap: 0.5rem;
  }
  .sidebar-user {
    gap: 0.5rem;
  }
  .user-badge .avatar {
    width: 36px;
    height: 36px;
    font-size: 1rem;
  }
  .nav-item {
    padding: 0.5rem 0.5rem;
    font-size: 0.85rem;
    gap: 0.5rem;
  }
  .logout {
    padding: 0.5rem 0.7rem;
    font-size: 0.9rem;
  }
  .top-bar {
    padding: 1rem 0.5rem 0.7rem;
    flex-direction: column;
    gap: 0.7rem;
  }
  .view-area {
    padding: 0 0.5rem 1rem;
  }
}
</style>
