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
            :title="item.label"
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
            <button @click="toggleTheme" class="ghost theme-toggle" :title="theme === 'dark' ? 'Светлая тема' : 'Темная тема'">
              <span v-if="theme === 'dark'">☀️</span>
              <span v-else>🌙</span>
            </button>
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
    const theme = ref(localStorage.getItem('theme') || 'dark')

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

    const toggleTheme = () => {
      theme.value = theme.value === 'dark' ? 'light' : 'dark'
      localStorage.setItem('theme', theme.value)
      document.documentElement.setAttribute('data-theme', theme.value)
    }

    const storageListener = (event) => {
      if (['defects', 'projects'].includes(event.key)) {
        refreshMetrics()
      }
    }

    onMounted(() => {
      refreshMetrics()
      window.addEventListener('storage', storageListener)
      document.documentElement.setAttribute('data-theme', theme.value)
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
      lastSyncLabel,
      theme,
      toggleTheme
    }
  }
}
</script>

<style>
@import url('https://fonts.googleapis.com/css2?family=Manrope:wght@400;600;700&display=swap');

:root {
  --transition: all 0.3s ease;
}

/* Dark Theme (default) */
:root,
[data-theme="dark"] {
  --bg-gradient-start: #1f2a40;
  --bg-gradient-mid: #0f1729;
  --bg-gradient-end: #05070c;
  --text-primary: #f6f7fb;
  --text-secondary: rgba(226, 232, 240, 0.86);
  --text-muted: rgba(148, 163, 184, 0.75);
  
  --card-bg: linear-gradient(180deg, rgba(17, 24, 39, 0.65) 0%, rgba(17, 24, 39, 0.35) 100%);
  --card-border: rgba(148, 163, 184, 0.1);
  --card-highlight-bg: linear-gradient(145deg, rgba(37, 99, 235, 0.35) 0%, rgba(14, 116, 144, 0.25) 100%);
  
  --sidebar-bg: linear-gradient(200deg, rgba(24, 33, 56, 0.95) 0%, rgba(12, 18, 31, 0.92) 100%);
  --content-bg: rgba(9, 13, 22, 0.7);
  
  --button-ghost-bg: rgba(96, 165, 250, 0.08);
  --button-ghost-border: rgba(96, 165, 250, 0.25);
  --button-ghost-text: #bfdbfe;
  --button-ghost-hover: rgba(96, 165, 250, 0.16);
  
  --accent-primary: #60a5fa;
  --accent-secondary: #38bdf8;
  --accent-success: #4ade80;
  --accent-warning: #facc15;
  --accent-danger: #fca5a5;
  
  --overlay-bg: rgba(2, 6, 23, 0.45);
  --shell-bg: rgba(10, 16, 28, 0.82);
  --shell-border: rgba(255, 255, 255, 0.08);
}

/* Light Theme */
[data-theme="light"] {
  --bg-gradient-start: #e0e7ff;
  --bg-gradient-mid: #f0f4ff;
  --bg-gradient-end: #ffffff;
  --text-primary: #0f172a;
  --text-secondary: rgba(15, 23, 42, 0.86);
  --text-muted: rgba(71, 85, 105, 0.75);
  
  --card-bg: linear-gradient(180deg, rgba(255, 255, 255, 0.95) 0%, rgba(248, 250, 252, 0.85) 100%);
  --card-border: rgba(203, 213, 225, 0.5);
  --card-highlight-bg: linear-gradient(145deg, rgba(99, 102, 241, 0.15) 0%, rgba(59, 130, 246, 0.1) 100%);
  
  --sidebar-bg: linear-gradient(200deg, rgba(248, 250, 252, 0.98) 0%, rgba(241, 245, 249, 0.95) 100%);
  --content-bg: rgba(255, 255, 255, 0.7);
  
  --button-ghost-bg: rgba(99, 102, 241, 0.08);
  --button-ghost-border: rgba(99, 102, 241, 0.25);
  --button-ghost-text: #4f46e5;
  --button-ghost-hover: rgba(99, 102, 241, 0.16);
  
  --accent-primary: #3b82f6;
  --accent-secondary: #0ea5e9;
  --accent-success: #22c55e;
  --accent-warning: #f59e0b;
  --accent-danger: #ef4444;
  
  --overlay-bg: rgba(15, 23, 42, 0.15);
  --shell-bg: rgba(255, 255, 255, 0.95);
  --shell-border: rgba(203, 213, 225, 0.5);
}

* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: 'Manrope', sans-serif;
  background: radial-gradient(120% 120% at 10% 10%, var(--bg-gradient-start) 0%, var(--bg-gradient-mid) 55%, var(--bg-gradient-end) 100%);
  color: var(--text-primary);
  min-height: 100vh;
  transition: var(--transition);
}

.app-root {
  min-height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: clamp(1rem, 3vw, 3rem) clamp(1rem, 4vw, 4rem);
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
  background: var(--card-bg);
  border: 1px solid var(--card-border);
  border-radius: 24px;
  padding: 2.5rem;
  backdrop-filter: blur(16px);
  box-shadow: 0 20px 60px var(--overlay-bg), 
              0 0 0 1px var(--card-border);
  transition: var(--transition);
  position: relative;
  overflow: hidden;
}

.landing-intro::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  height: 3px;
  background: linear-gradient(90deg, var(--accent-primary), var(--accent-secondary), var(--accent-success));
  opacity: 0.6;
}

.landing-intro h1 {
  font-size: 2.4rem;
  margin-bottom: 1rem;
  color: var(--text-primary);
  background: linear-gradient(135deg, var(--accent-primary), var(--accent-secondary));
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

.landing-intro p {
  color: var(--text-secondary);
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
  color: var(--text-secondary);
}

.landing-intro li::before {
  content: '•';
  color: var(--accent-primary);
  font-size: 1.4rem;
}

.app-shell {
  width: 100%;
  max-width: 1280px;
  display: grid;
  grid-template-columns: 280px 1fr;
  border-radius: 32px;
  border: 1px solid var(--shell-border);
  overflow: hidden;
  background: var(--shell-bg);
  box-shadow: 0 40px 70px var(--overlay-bg);
  backdrop-filter: blur(22px);
  transition: var(--transition);
}

.app-sidebar {
  background: var(--sidebar-bg);
  padding: 2.5rem 2rem;
  display: flex;
  flex-direction: column;
  gap: 2rem;
  transition: var(--transition);
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
  color: var(--text-muted);
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
  background: var(--button-ghost-bg);
  display: grid;
  place-items: center;
  font-weight: 700;
  color: var(--accent-primary);
  transition: var(--transition);
}

.user-name {
  font-weight: 600;
  color: var(--text-primary);
}

.user-role {
  font-size: 0.85rem;
  color: var(--text-muted);
}

.user-message {
  font-size: 0.9rem;
  color: var(--text-secondary);
}

.sidebar-nav {
  display: grid;
  gap: 0.5rem;
  flex: 1;
}

.nav-item {
  display: flex;
  align-items: center;
  gap: 1rem;
  padding: 0.85rem 1rem;
  border-radius: 14px;
  border: none;
  background: transparent;
  color: var(--text-secondary);
  font-size: 0.95rem;
  cursor: pointer;
  transition: var(--transition);
}

.nav-item:hover,
.nav-item.active {
  background: var(--button-ghost-bg);
  color: var(--text-primary);
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
  background: var(--button-ghost-bg);
  color: var(--text-secondary);
  border: 1px solid var(--button-ghost-border);
  border-radius: 12px;
  padding: 0.75rem 1rem;
  cursor: pointer;
  transition: var(--transition);
}

.logout:hover {
  background: var(--button-ghost-hover);
}

.app-content {
  background: var(--content-bg);
  display: grid;
  grid-template-rows: auto 1fr;
  backdrop-filter: blur(18px);
  transition: var(--transition);
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
  color: var(--text-primary);
}

.top-bar p {
  color: var(--text-muted);
  margin-top: 0.45rem;
  font-size: 0.9rem;
}

.theme-toggle {
  font-size: 1.3rem;
  padding: 0.5rem 0.8rem;
  min-width: 48px;
  display: inline-flex;
  align-items: center;
  justify-content: center;
}

.top-actions {
  display: flex;
  gap: 0.75rem;
}

.ghost {
  border: 1px solid var(--button-ghost-border);
  background: var(--button-ghost-bg);
  color: var(--button-ghost-text);
  border-radius: 12px;
  padding: 0.65rem 1.1rem;
  cursor: pointer;
  transition: var(--transition);
}

.ghost:hover {
  background: var(--button-ghost-hover);
}

.view-area {
  padding: 0 3rem 3rem;
  overflow-y: auto;
  min-height: calc(100vh - 280px);
  display: flex;
  flex-direction: column;
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
  background: var(--card-bg);
  border: 1px solid var(--card-border);
  border-radius: 24px;
  padding: 1.75rem;
  display: grid;
  gap: 1rem;
  box-shadow: inset 0 1px 0 var(--card-border);
  transition: var(--transition);
}

.pulse-card header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.pulse-card.highlight {
  grid-column: span 2;
  background: var(--card-highlight-bg);
}

.pulse-card h3 {
  color: var(--text-secondary);
  font-size: 1rem;
  letter-spacing: 0.4px;
  text-transform: uppercase;
}

.headline {
  font-size: 1.6rem;
  font-weight: 700;
  color: var(--text-primary);
}

.metric {
  font-size: 2.4rem;
  font-weight: 700;
  color: var(--accent-warning);
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
  background: linear-gradient(90deg, var(--accent-secondary) 0%, var(--accent-success) 100%);
  border-radius: inherit;
  transition: width 0.3s ease;
}

.caption {
  color: var(--text-muted);
  font-size: 0.9rem;
}

.overview-columns {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(260px, 1fr));
  gap: 1.5rem;
}

.badge {
  background: var(--button-ghost-bg);
  color: var(--button-ghost-text);
  border-radius: 1rem;
  padding: 0.2rem 0.75rem;
  font-size: 0.75rem;
  letter-spacing: 0.3px;
  transition: var(--transition);
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
  color: var(--text-primary);
}

.item-meta {
  font-size: 0.85rem;
  color: var(--text-muted);
  margin-top: 0.3rem;
}

.progress-line {
  display: grid;
  gap: 0.5rem;
  width: 140px;
}

.progress-line .line {
  height: 6px;
  background: var(--progress-bar-bg);
  border-radius: 999px;
  overflow: hidden;
}

.progress-line .fill {
  display: block;
  height: 100%;
  background: var(--progress-bar-fill);
}

.pill {
  border-radius: 999px;
  padding: 0.35rem 0.8rem;
  font-size: 0.8rem;
  font-weight: 600;
  letter-spacing: 0.2px;
}

.pill.danger { background: var(--pill-danger-bg); color: var(--accent-danger); }
.pill.warning { background: var(--pill-warning-bg); color: var(--accent-warning); }
.pill.good { background: var(--pill-good-bg); color: var(--accent-success); }

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
  background: var(--surface-border);
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
  background: linear-gradient(90deg, var(--accent-secondary) 0%, var(--accent-success) 100%);
  position: absolute;
  left: -1.4rem;
  top: 0.3rem;
}

.timeline small {
  color: var(--text-muted);
  font-size: 0.75rem;
}

.empty {
  color: var(--text-muted);
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
    padding: 1.5rem 1rem;
  }

  .sidebar-nav {
    grid-auto-flow: column;
    grid-auto-columns: 1fr;
    gap: 0.5rem;
  }

  .nav-item {
    padding: 0.6rem 0.8rem;
    font-size: 0.9rem;
    min-width: 80px;
    gap: 0.8rem;
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
    gap: 2rem;
  }

  .landing-intro {
    text-align: center;
    padding: 1rem;
  }

  .landing-intro h1 {
    font-size: 2.2rem;
  }

  .landing-intro ul {
    text-align: left;
    max-width: 400px;
    margin: 0 auto;
  }

  .app-root {
    padding: 1rem;
  }

  .app-sidebar {
    padding: 1rem;
    gap: 1rem;
  }

  .sidebar-user {
    display: none;
  }

  .brand {
    font-size: 1rem;
  }
}

@media (max-width: 640px) {
  .app-root {
    padding: 0.75rem;
  }

  .landing-intro {
    padding: 0.5rem;
  }

  .landing-intro h1 {
    font-size: 1.8rem;
  }

  .landing-intro p {
    font-size: 1rem;
  }

  .landing-intro ul {
    font-size: 0.9rem;
  }

  .app-sidebar {
    padding: 0.75rem;
    gap: 0.75rem;
  }

  .sidebar-user {
    gap: 0.6rem;
  }

  .user-badge .avatar {
    width: 40px;
    height: 40px;
    font-size: 1.1rem;
  }

  .user-name {
    font-size: 1rem;
  }

  .user-role {
    font-size: 0.85rem;
  }

  .nav-item {
    padding: 0.5rem 0.6rem;
    font-size: 0.85rem;
    gap: 1rem;
    min-width: 100px;
    border-radius: 10px;
  }

  .nav-icon {
    font-size: 1.1rem;
  }

  .logout {
    padding: 0.5rem 0.8rem;
    font-size: 0.9rem;
  }

  .top-bar {
    padding: 1rem 0.75rem;
    gap: 0.5rem;
  }

  .top-bar h2 {
    font-size: 1.5rem;
  }

  .top-bar p {
    font-size: 0.85rem;
  }

  .view-area {
    padding: 0 1rem 2rem;
  }
}

/* --- Мобильная адаптация --- */
@media (max-width: 480px) {
  .app-root {
    padding: 0.5rem;
  }

  .landing-intro {
    padding: 0.25rem;
  }

  .landing-intro h1 {
    font-size: 1.5rem;
  }

  .landing-intro p {
    font-size: 0.9rem;
  }

  .landing-intro ul {
    font-size: 0.85rem;
    padding-left: 1rem;
  }

  .app-shell {
    grid-template-columns: 1fr;
    gap: 1rem;
  }

  .app-sidebar {
    order: -1;
    min-width: 0;
    padding: 0.75rem 0.5rem;
    gap: 2rem;
    position: sticky;
    top: 0;
    z-index: 10;
    background: var(--sidebar-bg);
    border-radius: 12px;
    margin-bottom: 0;
    box-shadow: 0 4px 12px var(--overlay-bg);
  }

  .app-content {
    order: 1;
  }

  .sidebar-user {
    gap: 0.5rem;
  }

  .user-badge .avatar {
    width: 36px;
    height: 36px;
    font-size: 1rem;
  }

  .user-name {
    font-size: 0.95rem;
  }

  .user-role {
    font-size: 0.85rem;
  }

  .nav-item {
    padding: 0.5rem 1rem;
    font-size: 0.9rem;
    gap: 1rem;
    min-width: 100px;
    border-radius: 10px;
  }

  .nav-icon {
    font-size: 1.1rem;
  }

  .logout {
    padding: 0.5rem 0.8rem;
    font-size: 0.9rem;
  }

  .top-bar {
    padding: 1rem 0.5rem;
    flex-direction: column;
    gap: 0.75rem;
    align-items: stretch;
    background: var(--content-bg);
    border-radius: 12px;
    margin-bottom: 1rem;
    box-shadow: 0 2px 8px var(--overlay-bg);
  }

  .top-actions {
    justify-content: space-between;
    flex-direction: row;
  }

  .view-area {
    padding: 0 0.5rem 1rem;
  }

  .demo-button {
    width: 100%;
    justify-content: center;
  }
}

@media (max-width: 360px) {
  .app-root {
    padding: 0.25rem;
  }

  .landing-intro h1 {
    font-size: 1.3rem;
  }

  .landing-intro ul {
    font-size: 0.8rem;
  }

  .app-shell {
    gap: 0.75rem;
  }

  .app-sidebar {
    padding: 0.5rem 0.25rem;
    gap: 0.5rem;
  }

  .nav-item {
    padding: 0.4rem 0.5rem;
    font-size: 0.8rem;
    gap: 0.3rem;
    min-width: 70px;
    border-radius: 8px;
  }

  .nav-item span:not(.nav-icon):not(.nav-counter) {
    display: none;
  }

  .nav-icon {
    font-size: 1rem;
  }

  .top-bar {
    padding: 0.75rem 0.25rem;
    gap: 0.5rem;
  }

  .view-area {
    padding: 0 0.25rem 0.75rem;
  }
}
</style>
