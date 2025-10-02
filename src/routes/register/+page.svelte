<script>
  import { supabase } from '$lib/supabase'
  import { goto } from '$app/navigation'

  let email = ''
  let password = ''
  let username = ''
  let message = ''

  const register = async () => {
    message = "Processing..."

    const { data, error } = await supabase.auth.signUp({
      email,
      password,
      options: {
        data: { username }
      }
    })

    if (error) {
      console.error(error)
      message = error.message
      return
    }

    message = "Registrasi berhasil! Cek email kamu untuk verifikasi sebelum login."
  }
</script>

<div class="auth-container">
  <div class="auth-card">
    <div class="auth-header">
      <img src="/logo3.png" alt="Logo" class="logo" />
      <h2>Dialog Senja</h2>
      <p>Silakan register untuk melanjutkan</p>
    </div>

    <form on:submit|preventDefault={register} class="auth-form">
      <div class="input-group">
        <input type="text" placeholder="Username" bind:value={username} required />
      </div>
      <div class="input-group">
        <input type="email" placeholder="Email" bind:value={email} required />
      </div>
      <div class="input-group">
        <input type="password" placeholder="Password" bind:value={password} required />
      </div>

      <button type="submit">Register</button>
    </form>

    {#if message}
      <p class="message">{message}</p>
    {/if}

    <p class="switch" on:click={() => goto("/login")}>
      Sudah punya akun? <span>Login</span>
    </p>
  </div>
</div>

<style>
  body {
    margin: 0;
    font-family: Arial, sans-serif;
    background: linear-gradient(135deg, #4f46e5, #3b82f6);
  }

  .auth-container {
    display: flex;
    justify-content: center;
    align-items: center;
    min-height: 100vh;
  }

  .auth-card {
    background: #fff;
    padding: 2rem;
    width: 100%;
    max-width: 400px;
    border-radius: 12px;
    box-shadow: 0 6px 18px rgba(0,0,0,0.1);
    text-align: center;
  }

  .auth-header .logo {
    width: 60px;
    height: 60px;
    border-radius: 50%;
    object-fit: cover;
    margin-bottom: 0.5rem;
  }

  .auth-header h2 {
    margin: 0;
    font-size: 1.5rem;
    color: #1f2937;
  }

  .auth-header p {
    margin: 0.3rem 0 1.5rem;
    font-size: 0.9rem;
    color: #6b7280;
  }

  .auth-form {
    display: flex;
    flex-direction: column;
    gap: 1rem;
  }

  .input-group input {
    flex: 1;
    padding: 0.8rem;
    border: 1px solid #d1d5db;
    border-radius: 8px;
    font-size: 1rem;
    width: 100%;
  }

  .input-group input:focus {
    outline: none;
    border-color: #3b82f6;
    box-shadow: 0 0 0 2px rgba(59,130,246,0.3);
  }

  .auth-form button[type="submit"] {
    padding: 0.8rem;
    border: none;
    border-radius: 8px;
    background: #000000;
    color: #fff;
    font-size: 1rem;
    font-weight: bold;
    cursor: pointer;
    transition: background 0.3s;
  }

  .auth-form button[type="submit"]:hover {
    background: #535957;
  }

  .switch {
    margin-top: 1rem;
    font-size: 0.9rem;
    color: #374151;
  }

  .switch span {
    color: #10b981;
    cursor: pointer;
    font-weight: bold;
  }

  .message {
    margin-top: 1rem;
    color: #ef4444;
    font-size: 0.9rem;
  }
</style>
