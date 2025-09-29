<script>
  import { supabase } from "$lib/supabaseClient";
  import { goto } from "$app/navigation";

  let username = "";
  let email = "";
  let password = "";
  let role = "user";
  let message = "";

  async function handleRegister() {
    message = "";

    try {
      const { data: existing } = await supabase
        .from("users")
        .select("id")
        .eq("email", email)
        .single();

      if (existing) {
        message = "Email sudah terdaftar!";
        return;
      }

      const { error } = await supabase.from("users").insert([
        { username, email, password, role }
      ]);

      if (error) throw error;

      message = "Registrasi berhasil! Silakan login.";
      setTimeout(() => goto("/login"), 1500);
    } catch (err) {
      message = err instanceof Error ? err.message : String(err);
    }
  }
</script>

<div class="auth-container">
  <div class="auth-card">
    <div class="auth-header">
      <img src="/logo.png" alt="Logo" class="logo" />
      <h2>MyCompany</h2>
      <p>Buat akun baru untuk melanjutkan</p>
    </div>

    <form on:submit|preventDefault={handleRegister} class="auth-form">
      <input type="text" placeholder="Username" bind:value={username} required />
      <input type="email" placeholder="Email" bind:value={email} required />
      <input type="password" placeholder="Password" bind:value={password} required />

      <select bind:value={role}>
        <option value="user">User</option>
        <option value="admin">Admin</option>
      </select>

      <button type="submit">Register</button>
    </form>

    <p class="switch" on:click={() => goto("/login")}>
      Sudah punya akun? <span>Login</span>
    </p>

    {#if message}
      <p class="message">{message}</p>
    {/if}
  </div>
</div>

<style>
  body {
    margin: 0;
    font-family: Arial, sans-serif;
    background: linear-gradient(135deg, #10b981, #059669);
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

  .auth-form input,
  .auth-form select {
    padding: 0.8rem;
    border: 1px solid #d1d5db;
    border-radius: 8px;
    font-size: 1rem;
  }

  .auth-form input:focus,
  .auth-form select:focus {
    outline: none;
    border-color: #10b981;
    box-shadow: 0 0 0 2px rgba(16,185,129,0.3);
  }

  .auth-form button {
    padding: 0.8rem;
    border: none;
    border-radius: 8px;
    background: #10b981;
    color: #fff;
    font-size: 1rem;
    font-weight: bold;
    cursor: pointer;
    transition: background 0.3s;
  }

  .auth-form button:hover {
    background: #059669;
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
