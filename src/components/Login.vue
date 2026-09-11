<template>
  <div class="login-card">
    <h2>Iniciar Sesión</h2>
    <p class="subtitle">Plataforma Emprendedores Locales de Ñuble</p>

    <form @submit.prevent="handleLogin">
      <div class="form-group">
        <label for="email">Correo Electrónico:</label>
        <input
          id="email"
          type="email"
          v-model="email"
          placeholder="ejemplo@nuble.cl"
          required
        />
      </div>

      <div class="form-group">
        <label for="password">Contraseña:</label>
        <input
          id="password"
          type="password"
          v-model="password"
          placeholder="••••••••"
          required
        />
      </div>

      <div v-if="errorMessage" class="error-banner">
        {{ errorMessage }}
      </div>

      <button type="submit" :disabled="loading" class="btn-primary">
        {{ loading ? 'Ingresando...' : 'Iniciar Sesión' }}
      </button>
    </form>
  </div>
</template>

<script>
export default {
  name: 'Login',
  emits: ['login-success'],
  data() {
    return {
      email: '',
      password: '',
      errorMessage: '',
      loading: false
    };
  },
  methods: {
    async handleLogin() {
      this.errorMessage = '';
      this.loading = true;

      try {
        const res = await fetch('http://localhost:3000/auth/login', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({
            email: this.email,
            password: this.password
          })
        });

        const data = await res.json();

        if (!res.ok) {
          throw new Error(data.error || 'Credenciales inválidas');
        }

        // Guardar credenciales de sesión en localStorage
        localStorage.setItem('token', data.token);
        localStorage.setItem('user', JSON.stringify(data.usuario));

        // Notificar al componente padre
        this.$emit('login-success', data.usuario);
      } catch (err) {
        this.errorMessage = err.message;
      } finally {
        this.loading = false;
      }
    }
  }
};
</script>

<style scoped>
.login-card {
  max-width: 400px;
  margin: 40px auto;
  padding: 2rem;
  background: #ffffff;
  border-radius: 8px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
}
.subtitle {
  color: #666;
  font-size: 0.9rem;
  margin-bottom: 1.5rem;
}
.form-group {
  margin-bottom: 1.2rem;
  text-align: left;
}
label {
  display: block;
  font-weight: bold;
  margin-bottom: 0.4rem;
  color: #333;
}
input {
  width: 100%;
  padding: 0.6rem;
  box-sizing: border-box;
  border: 1px solid #ccc;
  border-radius: 4px;
}
.error-banner {
  background-color: #ffebee;
  color: #c62828;
  padding: 0.75rem;
  border-radius: 4px;
  margin-bottom: 1rem;
  font-size: 0.9rem;
}
.btn-primary {
  width: 100%;
  padding: 0.75rem;
  background-color: #2e7d32;
  color: white;
  border: none;
  border-radius: 4px;
  font-size: 1rem;
  cursor: pointer;
  font-weight: bold;
}
.btn-primary:hover {
  background-color: #1b5e20;
}
.btn-primary:disabled {
  background-color: #a5d6a7;
  cursor: not-allowed;
}
</style>