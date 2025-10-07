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
  background: var(--modal-bg);
  border-radius: 32px;
  border: 1px solid var(--card-border);
  padding: 2.5rem;
  display: grid;
  gap: 2rem;
  box-shadow: 0 30px 60px var(--overlay-bg);
  transition: var(--transition);
}

header h2 {
  font-size: 1.8rem;
  font-weight: 700;
  color: var(--text-primary);
}

header p {
  color: var(--text-muted);
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
  color: var(--text-light);
}

.field input {
  padding: 0.85rem 1rem;
  border-radius: 14px;
  border: 1px solid var(--input-border);
  background: var(--input-bg);
  color: var(--input-text);
  outline: none;
  transition: var(--transition);
}

.field input:focus {
  border-color: var(--input-border-focus);
  box-shadow: 0 0 0 2px var(--button-ghost-bg);
}

.remember {
  display: flex;
  align-items: center;
  gap: 0.6rem;
  font-size: 0.85rem;
  color: var(--text-muted);
}

button[type="submit"] {
  border: none;
  border-radius: 16px;
  padding: 0.95rem 1rem;
  background: var(--button-primary-bg);
  color: var(--button-primary-text);
  font-weight: 700;
  cursor: pointer;
  transition: var(--transition), transform 0.2s ease, box-shadow 0.2s ease;
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
  color: var(--accent-danger);
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
  color: var(--text-light);
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
  border: 1px solid var(--surface-border);
  background: var(--input-bg);
  color: var(--text-primary);
  cursor: pointer;
  transition: var(--transition);
}

.quick-buttons button:hover {
  background: var(--button-ghost-bg);
  border-color: var(--button-ghost-border);
}

.quick-buttons .role {
  font-weight: 600;
}

.quick-buttons .meta {
  font-size: 0.85rem;
  color: var(--text-muted);
}

@media (max-width: 640px) {
  .login-panel {
    padding: 1.5rem 1rem;
    max-width: 400px;
    margin: 0 auto;
  }

  .login-form {
    gap: 1rem;
  }

  .quick-login h3 {
    font-size: 1rem;
  }

  .quick-buttons {
    grid-template-columns: repeat(2, 1fr);
    gap: 0.5rem;
  }
}

@media (max-width: 480px) {
  .login-panel {
    padding: 1rem 0.5rem;
    border-radius: 16px;
  }

  .login-form .field input {
    font-size: 1rem;
    padding: 0.6rem 0.7rem;
  }

  .quick-buttons button {
    padding: 0.6rem 0.7rem;
    font-size: 0.97rem;
  }

  .error {
    font-size: 0.97rem;
    padding: 0.5rem 0.7rem;
    border-radius: 10px;
  }

  .quick-buttons {
    grid-template-columns: 1fr;
    gap: 0.4rem;
  }

  .quick-buttons button {
    width: 100%;
    justify-content: center;
  }
}

@media (max-width: 360px) {
  .login-panel {
    padding: 0.75rem 0.4rem;
  }

  .login-form {
    gap: 0.75rem;
  }

  .field label {
    font-size: 0.9rem;
  }

  .field input {
    font-size: 0.95rem;
    padding: 0.55rem 0.65rem;
  }

  .quick-login h3 {
    font-size: 0.9rem;
  }

  .quick-buttons button {
    font-size: 0.9rem;
    padding: 0.55rem 0.65rem;
  }

  .error {
    font-size: 0.9rem;
    padding: 0.4rem 0.6rem;
  }
}
</style>
