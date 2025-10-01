<script lang="ts">
  import { supabase } from "../../lib/supabaseClient";
  import { goto } from "$app/navigation";

  let customerName: string = "";
  let cart: { name: string; price: number; qty: number; image: string }[] = [];
  let message: string = "";
  let activeCategory: string = "Makanan";
  let showCart: boolean = false;
  let showSummary: boolean = false;
  let paymentMethod: "qris" | "cash" | "" = "";
  let qrisImage = "/qris.png";
  let showSidebar: boolean = false;
  let orderType: "dine_in" | "take_away" = "dine_in"; // default
  

  let qrisProofFile: File | null = null;     // file asli
  let qrisPreview: string | null = null;     // preview lokal
  let qrisProofUrl: string | null = null;    // URL dari Supabase

  const menu = [
    {
      category: "Makanan",
      items: [
        { name: "Nasi Goreng", price: 20000, image: "/Nasi Goreng Teri Pete.png", bestSeller: true, rating: 4.8 },
        { name: "Mie Goreng", price: 20000, image: "/miegoreng.png", bestSeller: false, rating: 4.2 },
        { name: "Mie rebus", price: 15000, image: "/mierebus.png", bestSeller: true, rating: 4.5 },
        { name: "Mie Ayam", price: 25000, image: "/Mie ayam.png", bestSeller: false, rating: 4.0 },
      ],
    },
    {
      category: "Minuman",
      items: [
        { name: "Es Teh", price: 5000, image: "/tehobeng.png", bestSeller: false, rating: 4.1 },
        { name: "Kopi Torabica", price: 10000, image: "https://images.unsplash.com/photo-1509042239860-f550ce710b93", bestSeller: true, rating: 4.9 },
        { name: "Kopi Dialog senja", price: 10000, image: "/logo3.png", bestSeller: true, rating: 4.9 },
      ],
    },
  ];

  let qtySelections: Record<string, number> = {};

  interface Item {
    name: string;
    price: number;
    image: string;
  }

  function goToLogin() {
    goto("/login");
  }

  let addedItems: Record<string, boolean> = {}; // menyimpan state tombol sudah diklik

function addToCart(item: Item) {
  const qty = qtySelections[item.name] || 1;
  if (qty <= 0) return;

  const existing = cart.find((c) => c.name === item.name);
  if (existing) existing.qty += qty;
  else cart = [...cart, { ...item, qty }];

  qtySelections[item.name] = 1;

  // tandai item sudah pernah ditambahkan
  addedItems[item.name] = true;
}

  function removeFromCart(name: string) {
    cart = cart.filter((c) => c.name !== name);
  }

  $: total = cart.reduce((sum, c) => sum + c.price * c.qty, 0);

async function handleCheckout() {
  if (!customerName || cart.length === 0 || !paymentMethod) {
    message = "Isi nama, pilih menu, dan metode pembayaran!";
    return;
  }

  if (paymentMethod === "qris") {
    if (!qrisProofFile) {
      message = "Upload bukti pembayaran QRIS terlebih dahulu!";
      return;
    }
  }

  try {
    qrisProofUrl = null;

    if (paymentMethod === "qris" && qrisProofFile) {
      const fileName = `${Date.now()}-${qrisProofFile.name}`;
      const { error: uploadError } = await supabase.storage
        .from("qris_proofs")
        .upload(fileName, qrisProofFile);

      if (uploadError) throw uploadError;

      qrisProofUrl = supabase.storage
        .from("qris_proofs")
        .getPublicUrl(fileName).data.publicUrl;
    }

    const { error } = await supabase.from("orders").insert([
  {
    customer_name: customerName,
    items: cart,
    total_price: total,
    status: "pending",
    payment_method: paymentMethod,
    order_type: orderType,   // ✅ sudah sesuai constraint
    qris_proof: qrisProofUrl,
  },
]);

    if (error) throw error;

    message = "Pesanan berhasil dikirim ke admin!";
    showSummary = true;
  } catch (err: unknown) {
    message = err instanceof Error ? err.message : String(err);
  }
}


  // render bintang rating
  function renderStars(rating: number) {
    const fullStars = Math.floor(rating);
    const halfStar = rating % 1 >= 0.5;
    const stars = "★".repeat(fullStars) + (halfStar ? "☆" : "");
    return stars.padEnd(5, "☆");
  }

  // handle preview file bukti QRIS
  function handleFileChange(e: Event) {
    const file = (e.target as HTMLInputElement).files?.[0] || null;
    qrisProofFile = file;
    if (file) {
      qrisPreview = URL.createObjectURL(file);
    } else {
      qrisPreview = null;
    }
  }
</script>

<!-- Topbar Mobile -->
<div class="topbar-mobile">
  <button class="burger" on:click={() => (showSidebar = true)}>☰</button>
  <span class="brand-mobile">Dialog Senja</span>
  <button class="cart-btn" on:click={() => (showCart = true)}>🛒 ({cart.length})</button>
</div>

<div class="layout">
  <!-- Sidebar -->
  <aside class="sidebar {showSidebar ? 'open' : ''}">
    <div class="brand">
      <img src="/logo3.png" alt="Logo" class="logo" />
      <span class="company-name">Dialog Senja</span>
    </div>

    <nav class="categories">
      {#each menu as group}
        <button
          class:selected={activeCategory === group.category}
          on:click={() => {
            activeCategory = group.category;
            showSidebar = false;
          }}
        >
          {group.category}
        </button>
      {/each}
      <button class="login-btn desktop" on:click={goToLogin}>🔑 Login</button>
    </nav>

    <button class="cart-btn desktop" on:click={() => (showCart = true)}>
      🛒 Keranjang ({cart.length})
    </button>
  </aside>

  <!-- Overlay sidebar mobile -->
  {#if showSidebar}
    <div class="sidebar-overlay" on:click={() => (showSidebar = false)}></div>
  {/if}

  <!-- Konten kanan -->
  <main class="content">
    <div class="menu-grid">
      {#each menu as group}
        {#if group.category === activeCategory}
          {#each group.items as m}
            <div class="card">
              <div class="img-wrapper">
                <img src={m.image} alt={m.name} />
                {#if m.bestSeller}
                  <span class="badge"> Best Seller</span>
                {/if}
              </div>
              <h3>{m.name}</h3>
              <p class="price">Rp {m.price.toLocaleString()}</p>
              <p class="rating">{renderStars(m.rating)} <span>{m.rating.toFixed(1)}</span></p>
              <input type="number" min="1" bind:value={qtySelections[m.name]} placeholder="1" />
              <button 
  on:click={() => addToCart(m)} 
  disabled={addedItems[m.name]}
>
  {addedItems[m.name] ? "✔ Ditambahkan" : "+ Tambah"}
</button>

            </div>
          {/each}
        {/if}
      {/each}
    </div>
  </main>
</div>

<!-- Modal Keranjang -->
{#if showCart}
  <div class="modal-overlay">
    <div class="modal">
      <h2>🛒 Keranjang</h2>

      {#if !showSummary}
        <input type="text" placeholder="Nama Pemesan" bind:value={customerName} />
      {/if}

      {#if cart.length > 0 && !showSummary}

     <!-- Pilihan Dine In / Take Away -->
<div class="order-type">
  <label class:selected={orderType === "dine_in"}>
    <input type="radio" bind:group={orderType} value="dine_in" />
    Makan di Sini
  </label>
  <label class:selected={orderType === "take_away"}>
    <input type="radio" bind:group={orderType} value="take_away" />
    Take Away
  </label>
</div>

        <div class="payment-method">
          <label><input type="radio" bind:group={paymentMethod} value="qris" /> QRIS</label>
          <label><input type="radio" bind:group={paymentMethod} value="cash" /> Cash</label>
        </div>

        <ul>
          {#each cart as c}
            <li>
              <span>{c.name} × {c.qty}</span>
              <span>Rp {(c.price * c.qty).toLocaleString()}</span>
              <button class="remove" on:click={() => removeFromCart(c.name)}>✖</button>
            </li>
          {/each}
        </ul>
        <p class="total">Total: Rp {total.toLocaleString()}</p>

        {#if paymentMethod === "qris"}
          <div class="qris-section">
            <p>Scan QRIS berikut untuk pembayaran:</p>
            <img src={qrisImage} alt="QRIS Pembayaran" class="qris-img" />
            <p>Upload bukti pembayaran:</p>
            <input type="file" accept="image/*" on:change={handleFileChange} />
            {#if qrisPreview}
              <p>Preview bukti pembayaran:</p>
              <img src={qrisPreview} alt="Preview Bukti QRIS" class="qris-preview" />
            {/if}
          </div>
        {/if}

        <button class="checkout" on:click={handleCheckout}>💳 Konfirmasi Pesanan</button>

      {:else if showSummary}
        <div class="order-summary">
          <h3>👤 Pemesan: {customerName}</h3>
          <ul>
            {#each cart as c}
              <li>
                <span>{c.name} × {c.qty}</span>
                <span>Rp {(c.price * c.qty).toLocaleString()}</span>
              </li>
            {/each}
          </ul>
          <p class="total">Total: Rp {total.toLocaleString()}</p>

          {#if paymentMethod === "qris"}
            <div class="qris-section">
              <p>Bukti pembayaran QRIS:</p>
              {#if qrisProofUrl}
                <img src={qrisProofUrl} alt="Bukti Pembayaran QRIS" class="qris-img" />
              {:else}
                <p><em>Bukti pembayaran belum tersedia.</em></p>
              {/if}
            </div>
          {:else if paymentMethod === "cash"}
            <p class="cash-note">💵 Silakan bayar tunai ke kasir.</p>
          {/if}
        </div>
      {:else}
        <p class="empty">Keranjang kosong</p>
      {/if}

      {#if message}
        <p class="msg">{message}</p>
      {/if}

      <button 
        class="close-btn" 
        on:click={() => { 
          showCart = false; 
          showSummary = false; 
          paymentMethod = ""; 
          cart = []; 
          customerName = ""; 
          qtySelections = {}; 
          qrisProofFile = null;
          qrisPreview = null;
          qrisProofUrl = null;
          addedItems = {};   // ✅ reset state tombol
        }}
      >
        Tutup
      </button>
    </div>
  </div>
{/if}

<style>
body {
  margin: 0;
  font-family: Arial, sans-serif;
  background: #f9fafb;
}

.layout {
  display: flex;
  min-height: 100vh;
}

/* Topbar Mobile */
.topbar-mobile {
  display: none;
  background: #00754a;
  color: white;
  padding: 10px 15px;
  justify-content: space-between;
  align-items: center;
}
.burger {
  background: none;
  border: none;
  font-size: 26px;
  color: white;
  cursor: pointer;
}
.brand-mobile { font-weight: bold; }
.topbar-mobile .cart-btn {
  background: #005f3b;
  padding: 6px 12px;
  border-radius: 8px;
  border: none;
  color: white;
  font-weight: bold;
  cursor: pointer;
}

/* Sidebar kiri */
.sidebar {
  width: 220px;
  background: #00754a;
  color: white;
  padding: 20px;
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  transition: transform 0.3s ease-in-out;
}
.brand {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-bottom: 30px;
}
.brand .logo { width: 50px; height: 50px; border-radius: 50%; object-fit: cover; }
.brand .company-name { font-weight: bold; font-size: 1.3rem; }
.categories { display: flex; flex-direction: column; gap: 15px; width: 100%; margin-bottom: auto; }
.categories button {
  width: 100%;
  padding: 10px 15px;
  border-radius: 12px;
  border: none;
  background: #00a66a;
  color: white;
  font-weight: 600;
  cursor: pointer;
  text-align: left;
}
.categories button.selected { background: #fff; color: #00754a; }
.cart-btn.desktop {
  margin-top: 30px;
  width: 100%;
  padding: 8px;
  border-radius: 12px;
  border: none;
  background: #005f3b;
  color: white;
  font-weight: bold;
  cursor: pointer;
}

/* Overlay sidebar mobile */
.sidebar-overlay {
  position: fixed;
  top: 0; left: 0; right: 0; bottom: 0;
  background: rgba(0,0,0,0.5);
  z-index: 900;
}

/* Konten kanan */
.content {
  flex: 1;
  padding: 20px;
  overflow-y: auto;
}
.menu-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(160px, 1fr));
  gap: 16px;
}
.card {
  position: relative;
  background: white;
  border-radius: 16px;
  padding: 16px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
  display: flex;
  flex-direction: column;
  align-items: center;
  text-align: center;
}
.img-wrapper { position: relative; width: 100%; }
.card img {
  width: 100%;
  height: 140px;
  object-fit: cover;
  border-radius: 12px;
  margin-bottom: 12px;
}
.badge {
  position: absolute;
  top: 8px;
  left: 8px;
  background: #ff5722;
  color: white;
  font-size: 0.7rem;
  font-weight: bold;
  padding: 4px 8px;
  border-radius: 6px;
}
.card h3 { margin: 0; font-size: 1rem; font-weight: 600; }
.card .price { color: #555; margin: 8px 0; }
.rating {
  font-size: 0.9rem;
  color: #f59e0b;
  margin: 4px 0;
}
.rating span {
  color: #555;
  margin-left: 4px;
  font-size: 0.8rem;
}
.card input { width: 60px; padding: 5px; border-radius: 8px; border: 1px solid #ccc; text-align: center; margin-bottom: 10px; }
.card button { background: #00754a; color: white; border: none; border-radius: 12px; padding: 10px; cursor: pointer; font-weight: 600; }

/* Modal */
.modal-overlay {
  position: fixed;
  top:0; left:0; right:0; bottom:0;
  background: rgba(0,0,0,0.5);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 999;
}
.modal {
  background:white;
  border-radius:16px;
  padding:20px;
  width:90%;
  max-width:500px;
  max-height:80vh;
  overflow-y:auto;
}
.modal input { width:90%; padding:10px; margin-bottom:12px; border:1px solid #ccc; border-radius:8px; }
.modal ul { list-style:none; padding:0; margin:0 0 15px; }
.modal li { display:flex; justify-content:space-between; align-items:center; padding:6px 0; border-bottom:1px solid #eee; }
.modal .remove { background:none; border:none; color:red; cursor:pointer; font-size:16px; }
.modal .total { font-weight:bold; margin:10px 0; }
.modal .checkout { width:100%; padding:12px; background:#00754a; color:white; border:none; border-radius:12px; font-size:16px; font-weight:bold; cursor:pointer; }
.modal .close-btn { margin-top:12px; width:100%; padding:10px; background:#ccc; border:none; border-radius:8px; cursor:pointer; }
.modal .msg { margin: 10px 0; color: #00754a; font-weight: bold; }

/* QRIS & Cash */
.payment-method {
  display: flex;
  gap: 20px;
  margin-bottom: 15px;
  font-weight: bold;
}
.cash-note {
  margin-top: 10px;
  font-weight: bold;
  color: #00754a;
}
.qris-section { text-align: center; margin: 15px 0; }
.qris-img {
  width: 250px;
  height: 250px;
  object-fit: contain;
  border-radius: 12px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.2);
}

/* Responsive */
@media (max-width: 800px) {
  .layout { flex-direction: column; }
  .topbar-mobile { display: flex; }

  .sidebar {
    position: fixed;
    top: 0; left: 0;
    height: 100%;
    z-index: 1000;
    transform: translateX(-100%);
  }
  .sidebar.open { transform: translateX(0); }
  .cart-btn.desktop { display: none; }

  .menu-grid {
    grid-template-columns: repeat(2, 1fr);
    gap: 12px;
  }
  .card img { height: 100px; }
}
/* ... semua style sama seperti punyamu, ditambah style preview ... */

.qris-preview {
  margin-top: 10px;
  width: 200px;
  border-radius: 8px;
  box-shadow: 0 2px 6px rgba(0,0,0,0.2);
}
.order-type {
  display: flex;
  justify-content: space-between;
  gap: 15px;
  margin: 15px 0;
}

.order-type label {
  flex: 1;
  padding: 10px;
  border: 2px solid #ccc;
  border-radius: 12px;
  text-align: center;
  font-weight: bold;
  cursor: pointer;
  transition: all 0.2s;
  background: #f9fafb;
}

.order-type label.selected {
  border-color: #00754a;
  background: #e6f9f0;
  color: #00754a;
}

.order-type input {
  display: none;
}

.order-type-summary {
  margin: 10px 0;
  font-size: 1.1rem;
  font-weight: 600;
  color: #00754a;
}
</style>


