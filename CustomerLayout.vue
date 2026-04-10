<template>
  <div class="customer-layout">
    <header class="navbar">
      <div class="nav-container">
        <router-link to="/" class="brand">
          <span class="logo-icon">☕</span>
          <h1>PolyCafe</h1>
        </router-link>

        <nav class="nav-links">
          <router-link to="/" class="nav-item">Thực đơn</router-link>
          
          <router-link v-if="user" to="/customer/orders" class="nav-item">
            📦 Đơn hàng
          </router-link>
          
          <router-link to="/cart" class="nav-item cart-link">
            🛒 Giỏ hàng
            <span class="cart-badge" v-if="cartCount > 0">{{ cartCount }}</span>
          </router-link>

          <router-link v-if="!user" to="/login" class="nav-item btn-login">Đăng nhập</router-link>

          <div v-else class="user-menu">
            <router-link v-if="user.role == 2" to="/admin" class="btn-back-admin">⚙️ Quản Trị</router-link>
            <router-link v-if="user.role == 1" to="/staff/pos" class="btn-back-admin">💻 Máy POS</router-link>

            <router-link to="/profile" class="nav-item user-profile">
              👤 {{ user.fullName || 'Hồ sơ' }}
            </router-link>
            <button @click="handleLogout" class="btn-logout">Đăng xuất</button>
          </div>
        </nav>
      </div>
    </header>

    <main class="main-content">
      <router-view />
    </main>

    <footer class="footer">
      <div class="footer-content">
        <p><strong>☕ PolyCafe</strong> - Thức uống đậm đà, lan tỏa niềm vui!</p>
        <p class="copyright">&copy; 2026 Bản quyền thuộc về PolyCafe. Hệ thống đặt món trực tuyến.</p>
      </div>
    </footer>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue';
import { useRouter } from 'vue-router';

const router = useRouter();
const user = ref(null);
const cartCount = ref(0);

// ================= NÂNG CẤP: ĐẾM ĐÚNG TỔNG SỐ LY NƯỚC =================
const updateCartCount = () => {
  const cart = JSON.parse(localStorage.getItem('cart')) || [];
  // Cộng dồn thuộc tính quantity của từng món trong giỏ
  cartCount.value = cart.reduce((total, item) => total + item.quantity, 0); 
};

const checkUser = () => {
  user.value = JSON.parse(localStorage.getItem('user'));
};

onMounted(() => {
  checkUser();
  updateCartCount();
  window.addEventListener('cart-updated', updateCartCount);
});

onUnmounted(() => {
  window.removeEventListener('cart-updated', updateCartCount);
});

const handleLogout = () => {
  if (confirm("Bạn có chắc chắn muốn đăng xuất khỏi tài khoản?")) {
    localStorage.removeItem('user');
    user.value = null;
    router.push('/login');
  }
};
</script>

<style scoped>
.customer-layout { display: flex; flex-direction: column; min-height: 100vh; background: #fdfbf7; }

/* NAVBAR STYLES */
.navbar { background: #8B4513; color: white; position: sticky; top: 0; z-index: 100; box-shadow: 0 4px 10px rgba(0,0,0,0.15); }
.nav-container { max-width: 1200px; margin: 0 auto; padding: 15px 20px; display: flex; justify-content: space-between; align-items: center; }

.brand { display: flex; align-items: center; text-decoration: none; color: #f1c40f; gap: 10px;}
.logo-icon { font-size: 28px; }
.brand h1 { margin: 0; font-size: 24px; font-weight: 800; letter-spacing: 1px; }

.nav-links { display: flex; align-items: center; gap: 25px; }
.nav-item { color: #fdfbf7; text-decoration: none; font-size: 16px; font-weight: 600; transition: 0.3s; position: relative;}
.nav-item:hover { color: #f39c12; }

/* Trạng thái Menu đang được chọn */
.router-link-active:not(.brand) { color: #f1c40f; }
.router-link-active:not(.brand)::after { content: ''; position: absolute; bottom: -5px; left: 0; width: 100%; height: 2px; background: #f1c40f; }

/* Nút Đăng Nhập / Đăng Xuất */
.btn-login { background: #f39c12; color: #fff !important; padding: 8px 20px; border-radius: 20px; border: 2px solid #f39c12; }
.btn-login:hover { background: transparent; color: #f39c12 !important; }
.btn-logout { background: transparent; color: #e74c3c; border: 1px solid #e74c3c; padding: 6px 15px; border-radius: 20px; cursor: pointer; font-weight: bold; transition: 0.3s;}
.btn-logout:hover { background: #e74c3c; color: white; }

.user-menu { display: flex; align-items: center; gap: 15px; }
.user-profile { color: #ecf0f1; border-right: 1px solid rgba(255,255,255,0.2); padding-right: 15px;}

/* NÚT CHUYỂN ĐỔI NHANH CHO ADMIN / NHÂN VIÊN */
.btn-back-admin { background: #27ae60; color: white !important; padding: 6px 15px; border-radius: 20px; text-decoration: none; font-weight: bold; font-size: 14px; transition: 0.3s; border: 2px solid transparent;}
.btn-back-admin:hover { background: #2ecc71; border-color: white; transform: translateY(-1px);}

/* Giỏ hàng và Badge */
.cart-link { display: flex; align-items: center; position: relative; }
.cart-badge { position: absolute; top: -10px; right: -15px; background: #e74c3c; color: white; font-size: 12px; font-weight: bold; width: 20px; height: 20px; display: flex; align-items: center; justify-content: center; border-radius: 50%; box-shadow: 0 2px 4px rgba(0,0,0,0.2); }

/* MAIN CONTENT */
.main-content { flex: 1; width: 100%; max-width: 1200px; margin: 0 auto; padding: 30px 20px; }

/* FOOTER */
.footer { background: #2c3e50; color: #bdc3c7; text-align: center; padding: 30px 20px; margin-top: auto;}
.footer-content p { margin: 5px 0; font-size: 15px; }
.footer-content strong { color: #f1c40f; }
.copyright { font-size: 13px; margin-top: 15px !important; opacity: 0.7; }

/* Responsive */
@media (max-width: 768px) {
  .nav-links { gap: 15px; }
  .user-profile { display: none; }
  .btn-back-admin { display: none; }
}
</style>