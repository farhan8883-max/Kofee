<script lang="ts">
  import { supabase } from "../../lib/supabaseClient";
  import { onMount } from "svelte";

  type Order = {
    id: number;
    customer_name: string;
    items: { name: string; price: number; qty: number }[];
    total_price: number;
    status: string;
    payment_method: "qris" | "cash";
    created_at: string;
  };

  type Summary = {
    period: string;
    label: string;
    total: number;
  };

  let orders: Order[] = [];
  let message: string = "";

  let dailySummary: Summary[] = [];
  let weeklySummary: Summary[] = [];
  let monthlySummary: Summary[] = [];
  let totalIncome: number = 0;

  let activeTab: "orders" | "summary" = "orders";
  let sidebarOpen = false;

  async function fetchOrders() {
    const { data, error } = await supabase
      .from("orders")
      .select("id, customer_name, items, total_price, status, payment_method, created_at")
      .order("id", { ascending: false });

    if (error) {
      message = error.message;
      return;
    }
    orders = ((data as Order[]) || []).filter(o => o.status !== "deleted");
  }

  async function markAsDone(id: number) {
    const { error } = await supabase.from("orders").update({ status: "done" }).eq("id", id);
    if (error) {
      message = error.message;
      return;
    }
    await fetchOrders();
    await fetchSummary();
  }

  async function deleteOrder(id: number) {
    const { error } = await supabase.from("orders").update({ status: "deleted" }).eq("id", id);
    if (error) {
      message = error.message;
      return;
    }
    await fetchOrders();
  }

  async function fetchSummary() {
    const { data, error } = await supabase
      .from("orders")
      .select("total_price, created_at, status");

    if (error) {
      message = error.message;
      return;
    }

    const byDay: Record<string, number> = {};
    const byWeek: Record<string, number> = {};
    const byMonth: Record<string, number> = {};
    totalIncome = 0;

    data?.forEach((o: any) => {
      if (o.status === "deleted") return;
      const d = new Date(o.created_at);
      totalIncome += o.total_price;

      const dayKey = d.toISOString().split("T")[0];
      byDay[dayKey] = (byDay[dayKey] || 0) + o.total_price;

      const week = getISOWeek(d);
      const weekKey = `${d.getFullYear()}-W${week}`;
      byWeek[weekKey] = (byWeek[weekKey] || 0) + o.total_price;

      const monthKey = `${d.getFullYear()}-${String(d.getMonth() + 1).padStart(2, "0")}`;
      byMonth[monthKey] = (byMonth[monthKey] || 0) + o.total_price;
    });

    dailySummary = Object.entries(byDay)
      .sort(([a], [b]) => a.localeCompare(b))
      .map(([period, total]) => ({ period, label: period, total }));

    weeklySummary = Object.entries(byWeek)
      .sort(([a], [b]) => a.localeCompare(b))
      .map(([period, total]) => {
        const [year, weekStr] = period.split("-W");
        return { period, label: `Minggu ke-${weekStr} (${year})`, total };
      });

    monthlySummary = Object.entries(byMonth)
      .sort(([a], [b]) => a.localeCompare(b))
      .map(([period, total]) => {
        const [year, month] = period.split("-");
        return { period, label: `${getMonthName(parseInt(month))} ${year}`, total };
      });
  }

  function getISOWeek(d: Date) {
    const date = new Date(Date.UTC(d.getFullYear(), d.getMonth(), d.getDate()));
    const dayNum = date.getUTCDay() || 7;
    date.setUTCDate(date.getUTCDate() + 4 - dayNum);
    const yearStart = new Date(Date.UTC(date.getUTCFullYear(), 0, 1));
    return Math.ceil(((date.getTime() - yearStart.getTime()) / 86400000 + 1) / 7);
  }

  function getMonthName(month: number) {
    const months = [
      "Januari","Februari","Maret","April","Mei","Juni",
      "Juli","Agustus","September","Oktober","November","Desember"
    ];
    return months[month - 1];
  }

  onMount(async () => {
    await fetchOrders();
    await fetchSummary();
  });

  async function logout() {
  const { error } = await supabase.auth.signOut();
  if (error) {
    message = error.message;
    return;
  }
  // redirect ke halaman login, misalnya /login
  window.location.href = "/login";
}

</script>

<div class="dashboard">
  <!-- Sidebar -->
<aside class:open={sidebarOpen} class="sidebar">
  <h2 class="logo">
    <img src="/logo3.png" alt="Logo" class="logo-img" />
    Cofee Street
  </h2>
  <nav>
    <button class:active={activeTab === "orders"} on:click={() => {activeTab = "orders"; sidebarOpen=false}}>📦 Pesanan</button>
    <button class:active={activeTab === "summary"} on:click={() => {activeTab = "summary"; sidebarOpen=false}}>💰 Ringkasan</button>
     <!-- Tombol Logout -->
  <button class="logout-btn" on:click={logout}>🚪 Logout</button>
  </nav>
 
</aside>


  <!-- Overlay untuk HP -->
  {#if sidebarOpen}
    <div class="overlay" on:click={() => sidebarOpen = false}></div>
  {/if}

  <!-- Main Content -->
  <main class="main">
    <header class="header">
      <button class="burger" on:click={() => sidebarOpen = !sidebarOpen}>☰</button>
      <h1>{activeTab === "orders" ? "Daftar Pesanan" : "Ringkasan Pemasukan"}</h1>
    </header>

    {#if message}
      <p class="msg">{message}</p>
    {/if}

    <!-- Orders -->
    {#if activeTab === "orders"}
      {#if orders.length > 0}
        <div class="table-wrapper">
          <table class="orders-table">
            <thead>
              <tr>
                <th>ID</th><th>Customer</th><th>Items</th><th>Total</th><th>Metode</th><th>Status</th><th>Aksi</th>
              </tr>
            </thead>
            <tbody>
              {#each orders as o}
                <tr>
                  <td>{o.id}</td>
                  <td>{o.customer_name}</td>
                  <td>
                    <ul>
                      {#each o.items as it}
                        <li>{it.name} × {it.qty}</li>
                      {/each}
                    </ul>
                  </td>
                  <td>Rp {o.total_price.toLocaleString()}</td>
                  <td><span class="tag {o.payment_method}">{o.payment_method}</span></td>
                  <td><span class="status {o.status}">{o.status}</span></td>
                  <td>
                    {#if o.status === "pending"}
                      <button class="done-btn" on:click={() => markAsDone(o.id)}>✔</button>
                    {:else}
                      <span class="done">✅</span>
                    {/if}
                    <button class="delete-btn" on:click={() => deleteOrder(o.id)}>🗑</button>
                  </td>
                </tr>
              {/each}
            </tbody>
          </table>
        </div>
      {:else}
        <p class="empty">Tidak ada pesanan.</p>
      {/if}
    {/if}

    <!-- Summary -->
    {#if activeTab === "summary"}
      <div class="summary">
        <p class="grand-total">Total Keseluruhan: <strong>Rp {totalIncome.toLocaleString()}</strong></p>

        <h3>Rekap Harian</h3>
        <div class="table-wrapper">
          <table class="summary-table">
            <thead><tr><th>Tanggal</th><th>Total</th></tr></thead>
            <tbody>
              {#each dailySummary as s}
                <tr><td>{s.label}</td><td>Rp {s.total.toLocaleString()}</td></tr>
              {/each}
            </tbody>
          </table>
        </div>

        <h3>Rekap Mingguan</h3>
        <ul class="weekly-list">
          {#each weeklySummary as s}
            <li>{s.label}: <strong>Rp {s.total.toLocaleString()}</strong></li>
          {/each}
        </ul>

        <h3>Rekap Bulanan</h3>
        <div class="table-wrapper">
          <table class="summary-table">
            <thead><tr><th>Bulan</th><th>Total</th><th>Perbandingan</th></tr></thead>
            <tbody>
              {#each monthlySummary as s, i}
                <tr>
                  <td>{s.label}</td>
                  <td>Rp {s.total.toLocaleString()}</td>
                  <td>
                    {#if i > 0}
                      {#if s.total - monthlySummary[i-1].total >= 0}
                        🔼 Rp {(s.total - monthlySummary[i-1].total).toLocaleString()}
                      {:else}
                        🔽 Rp {Math.abs(s.total - monthlySummary[i-1].total).toLocaleString()}
                      {/if}
                    {:else}
                      -
                    {/if}
                  </td>
                </tr>
              {/each}
            </tbody>
          </table>
        </div>
      </div>
    {/if}
  </main>
</div>

<style>
  * { box-sizing: border-box; margin: 0; padding: 0; }
  body { font-family: "Segoe UI", sans-serif; }

  .dashboard {
    display: flex;
    height: 100vh;
    background: #f9fafc;
    color: #333;
  }

  .sidebar {
    width: 220px;
    background: #fff;
    border-right: 1px solid #eee;
    padding: 20px;
    display: flex;
    flex-direction: column;
    gap: 20px;
    transition: transform 0.3s ease;
  }
  .sidebar .logo { font-size: 20px; font-weight: bold; color: #000000; }
  .sidebar nav button {
    padding: 10px 15px;
    border: none;
    background: none;
    text-align: left;
    cursor: pointer;
    font-size: 15px;
    border-radius: 8px;
  }
  .sidebar nav button.active, .sidebar nav button:hover {
    background: #4caf50;
    color: #fff;
  }

  .overlay {
    position: fixed;
    top:0; left:0; right:0; bottom:0;
    background: rgba(0,0,0,0.4);
    z-index: 9;
  }

  .main {
    flex: 1;
    padding: 20px 30px;
    overflow-y: auto;
    width: 100%;
  }

  .header {
    display: flex;
    align-items: center;
    gap: 15px;
    margin-bottom: 20px;
  }
  .header h1 { font-size: 22px; }
  .burger {
    display: none;
    font-size: 22px;
    background: none;
    border: none;
    cursor: pointer;
  }

  .table-wrapper { overflow-x: auto; }
  .orders-table, .summary-table {
    width: 100%;
    border-collapse: collapse;
    margin-bottom: 20px;
    background: #fff;
    border-radius: 8px;
    box-shadow: 0 2px 6px rgba(0,0,0,0.05);
    min-width: 600px;
  }
  th, td {
    padding: 12px 14px;
    border-bottom: 1px solid #eee;
    text-align: left;
    font-size: 14px;
  }
  th { background: #f4f4f8; }

  .tag {
    padding: 4px 8px;
    border-radius: 6px;
    font-size: 12px;
    color: #fff;
  }
  .tag.qris { background: #3f51b5; }
  .tag.cash { background: #ff9800; }

  .status.pending { color: #ff9800; font-weight: bold; }
  .status.done { color: #4caf50; font-weight: bold; }
  .done { color: #4caf50; }

  .done-btn, .delete-btn {
    padding: 5px 8px;
    border: none;
    border-radius: 6px;
    cursor: pointer;
    font-size: 13px;
    margin-right: 4px;
  }
  .done-btn { background: #4caf50; color: #fff; }
  .done-btn:hover { background: #3e9142; }
  .delete-btn { background: #e53935; color: #fff; }
  .delete-btn:hover { background: #c62828; }

  .weekly-list { list-style: none; padding: 0; }
  .weekly-list li { margin: 5px 0; }

  .grand-total { font-size: 16px; margin: 15px 0; }
  .msg { color: red; margin-bottom: 10px; }
  .empty { text-align: center; margin: 20px 0; font-style: italic; }

  /* Responsive */
  @media (max-width: 768px) {
    .sidebar {
      position: fixed;
      top:0; left:0;
      height: 100%;
      transform: translateX(-100%);
      z-index: 10;
    }
    .sidebar.open { transform: translateX(0); }
    .burger { display: block; }
    .main { padding: 15px; }
    .header h1 { font-size: 18px; }
    th, td { font-size: 13px; padding: 8px; }
  }
  .sidebar .logo {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 20px;
  font-weight: bold;
  color: #000000;
}

.logo-img {
  width: 28px;
  height: 28px;
  object-fit: contain;
}
.logout-btn {
  margin-top: auto; /* posisi di bawah sidebar */
  padding: 10px 15px;
  border: none;
  border-radius: 8px;
  background: #e53935;
  color: #000000;
  font-size: 15px;
  cursor: pointer;
}
.logout-btn:hover {
  background: #c62828;
}

</style>
