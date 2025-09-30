<template>
  <div class="login-shell">
    <div class="login-panel">
      <header>
        <h2>Вход в Pulse QA</h2>
        <p>Выберите тестовый аккаунт или введите данные вручную</p>
      </header>

      <form @submit.prevent="handleSubmit()" class="login-form">
        <label class="field">
          <span>Email</span>
          <input
            v-model="email"
            type="email"
            required
            placeholder="user@example.com"
            autocomplete="email"
          >
        </label>

        <label class="field">
          <span>Пароль</span>
          <input
            v-model="password"
            type="password"
            required
            placeholder="Введите пароль"
            autocomplete="current-password"
          >
        </label>

        <label class="remember">
          <input v-model="rememberMe" type="checkbox">
          <span>Запомнить меня на этом устройстве</span>
        </label>

        <button type="submit" :disabled="loading">
          {{ loading ? 'Проверяем данные…' : 'Войти в систему' }}
        </button>
      </form>

      <p v-if="error" class="error">{{ error }}</p>

      <section class="quick-login">
        <h3>Быстрый вход</h3>
        <div class="quick-buttons">
          <button
            v-for="account in quickAccounts"
            :key="account.email"
            type="button"
            @click="handleQuickLogin(account)"
          >
            <span class="role">{{ account.label }}</span>
            <span class="meta">{{ account.email }}</span>
          </button>
        </div>
      </section>
    </div>
  </div>
</template>

<script>
import { onMounted, ref } from 'vue'

export default {
  name: 'LoginView',
  emits: ['login'],
  setup(props, { emit }) {
    const email = ref('')
    const password = ref('')
    const rememberMe = ref(false)
    const loading = ref(false)
    const error = ref('')

    const users = {
      'engineer@test.com': {
        id: 1,
        email: 'engineer@test.com',
        name: 'Инженер Иванов',
        role: 'engineer',
        password: '123'
      },
      'manager@test.com': {
        id: 2,
        email: 'manager@test.com',
        name: 'Менеджер Петров',
        role: 'manager',
        password: '123'
      },
      'admin@test.com': {
        id: 3,
        email: 'admin@test.com',
        name: 'Руководитель Сидоров',
        role: 'admin',
        password: '123'
      }
    }

    const quickAccounts = [
      { label: 'Инженер', email: 'engineer@test.com', password: '123' },
      { label: 'Менеджер', email: 'manager@test.com', password: '123' },
      { label: 'Руководитель', email: 'admin@test.com', password: '123' }
    ]

    const handleSubmit = (fast = false) => {
      if (loading.value) return

      loading.value = true
      error.value = ''

      setTimeout(() => {
        const key = email.value.trim().toLowerCase()
        const user = users[key]

        if (user && user.password === password.value) {
          const { password: _, ...userWithoutPassword } = user

          if (rememberMe.value) {
            localStorage.setItem('lastUser', user.email)
          } else {
            localStorage.removeItem('lastUser')
          }

          emit('login', userWithoutPassword)
        } else {
          error.value = 'Неверный email или пароль'
        }

        loading.value = false
      }, fast ? 250 : 650)
    }

    const handleQuickLogin = (account) => {
      email.value = account.email
      password.value = account.password
      rememberMe.value = true
      handleSubmit(true)
    }

    onMounted(() => {
      const lastUser = localStorage.getItem('lastUser')
      if (lastUser && users[lastUser]) {
        email.value = lastUser
        rememberMe.value = true
      }
    })

    return {
      email,
      password,
      rememberMe,
      loading,
      error,
      quickAccounts,
      handleSubmit,
      handleQuickLogin
    }
  }
}
</script>

<style scoped>
.login-shell {
  width: 100%;
  display: flex;
  justify-content: center;
}

.login-panel {
  width: 100%;
  background: linear-gradient(180deg, rgba(15, 23, 42, 0.9) 0%, rgba(15, 23, 42, 0.6) 100%);
  border-radius: 32px;
  border: 1px solid rgba(148, 163, 184, 0.15);
  padding: 2.5rem;
  display: grid;
  gap: 2rem;
  box-shadow: 0 30px 60px rgba(2, 6, 23, 0.55);
}

header h2 {
  font-size: 1.8rem;
  font-weight: 700;
  color: #e2e8f0;
}

header p {
  color: rgba(148, 163, 184, 0.75);
  margin-top: 0.65rem;
  font-size: 0.95rem;
}

.login-form {
  display: grid;
  gap: 1.25rem;
}

.field {
  display: grid;
  gap: 0.5rem;
  font-size: 0.9rem;
  color: rgba(203, 213, 225, 0.86);
}

.field input {
  padding: 0.85rem 1rem;
  border-radius: 14px;
  border: 1px solid rgba(148, 163, 184, 0.2);
  background: rgba(15, 23, 42, 0.65);
  color: #f8fafc;
  outline: none;
  transition: border 0.2s ease, box-shadow 0.2s ease;
}

.field input:focus {
  border-color: rgba(96, 165, 250, 0.45);
  box-shadow: 0 0 0 2px rgba(59, 130, 246, 0.25);
}

.remember {
  display: flex;
  align-items: center;
  gap: 0.6rem;
  font-size: 0.85rem;
  color: rgba(148, 163, 184, 0.75);
}

button[type="submit"] {
  border: none;
  border-radius: 16px;
  padding: 0.95rem 1rem;
  background: linear-gradient(135deg, #38bdf8 0%, #6366f1 100%);
  color: #0f172a;
  font-weight: 700;
  cursor: pointer;
  transition: transform 0.2s ease, box-shadow 0.2s ease;
}

button[type="submit"]:hover:not(:disabled) {
  transform: translateY(-2px);
  box-shadow: 0 12px 24px rgba(99, 102, 241, 0.35);
}

button[type="submit"]:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.error {
  color: #f87171;
  background: rgba(248, 113, 113, 0.12);
  border-radius: 14px;
  padding: 0.75rem 1rem;
  border: 1px solid rgba(248, 113, 113, 0.25);
  text-align: center;
}

.quick-login {
  display: grid;
  gap: 1rem;
}

.quick-login h3 {
  font-size: 1rem;
  text-transform: uppercase;
  letter-spacing: 0.3px;
  color: rgba(203, 213, 225, 0.8);
}

.quick-buttons {
  display: grid;
  gap: 0.8rem;
}

.quick-buttons button {
  display: flex;
  justify-content: space-between;
  padding: 0.85rem 1.2rem;
  border-radius: 14px;
  border: 1px solid rgba(96, 165, 250, 0.18);
  background: rgba(15, 23, 42, 0.55);
  color: #e2e8f0;
  cursor: pointer;
  transition: all 0.2s ease;
}

.quick-buttons button:hover {
  background: rgba(37, 99, 235, 0.28);
  border-color: rgba(37, 99, 235, 0.35);
}

.quick-buttons .role {
  font-weight: 600;
}

.quick-buttons .meta {
  font-size: 0.85rem;
  color: rgba(148, 163, 184, 0.7);
}

@media (max-width: 640px) {
  .login-panel {
    padding: 2rem 1.5rem;
  }
}
</style>
