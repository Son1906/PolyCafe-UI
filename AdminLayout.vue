<template>
  <div class="admin-layout">
    <aside class="sidebar">
      <div class="brand">
        <h2>👑 PolyCafe</h2>
        <p class="role-text">Quản trị hệ thống</p>
      </div>

      <nav class="menu">
        <p class="group-title">QUẢN LÝ KINH DOANH</p>
        <router-link to="/admin/dashboard" class="menu-item">📊 Thống kê doanh thu</router-link>
        <router-link to="/admin/bills" class="menu-item">🧾 Quản lý Hóa đơn</router-link>
        <router-link to="/admin/reviews" class="menu-item">⭐ Quản lý Đánh giá</router-link>
        
        <p class="group-title">QUẢN LÝ THỰC ĐƠN</p>
        <router-link to="/admin/drinks" class="menu-item">☕ Quản lý Đồ uống</router-link>
        <router-link to="/admin/categories" class="menu-item">📁 Quản lý Danh mục</router-link>
        <router-link to="/admin/toppings" class="menu-item">🧋 Quản lý Topping</router-link>
        
        <p class="group-title">KHO & KHUYẾN MÃI</p>
        <router-link to="/admin/inventory" class="menu-item">📦 Tồn kho Nguyên liệu</router-link>
        <router-link to="/admin/receipts" class="menu-item">📋 Phiếu Nhập kho</router-link>
        <router-link to="/admin/vouchers" class="menu-item">🎟️ Quản lý Voucher</router-link>
        
        <p class="group-title">HỆ THỐNG</p>
        <router-link to="/admin/users" class="menu-item">👥 Quản lý Tài khoản</router-link>

        <p class="group-title">CHUYỂN ĐỔI NHANH</p>
        <router-link to="/" class="menu-item pos-link">🏠 Về Trang Khách Hàng</router-link>
        <router-link to="/staff/pos" class="menu-item pos-link">💻 Sang Máy Bán Hàng</router-link>
        <router-link to="/staff/kds" class="menu-item pos-link">🍳 Sang Màn Hình Bếp</router-link>
      </nav>

      <div class="sidebar-footer">
        <router-link to="/admin/profile" class="profile-btn">👤 Hồ Sơ Của Tôi</router-link>
        <button @click="handleLogout" class="logout-btn">👋 Đăng Xuất</button>
      </div>
    </aside>

    <main class="main-content">
      <router-view />
    </main>
  </div>
</template>

<script setup>
import { useRouter } from 'vue-router';

const router = useRouter();

// Logic Đăng xuất trực tiếp từ thanh Menu
const handleLogout = () => {
  if (confirm("Bạn có chắc chắn muốn đăng xuất khỏi hệ thống quản trị?")) {
    localStorage.removeItem('user'); // Xóa phiên đăng nhập
    router.push('/login');           // Đẩy về trang Login
  }
};
</script>

<style scoped>
.admin-layout { display: flex; height: 100vh; background: #f4f6f8; }

/* SIDEBAR STYLE */
.sidebar { width: 260px; background: #2c3e50; color: white; display: flex; flex-direction: column; box-shadow: 2px 0 5px rgba(0,0,0,0.1); z-index: 10; }
.brand { padding: 25px 20px; background: #1a252f; text-align: center; border-bottom: 1px solid #34495e; }
.brand h2 { color: #f39c12; margin: 0 0 5px 0; font-size: 24px;}
.role-text { color: #95a5a6; font-size: 13px; margin: 0; }

.menu { flex: 1; padding: 20px 0; overflow-y: auto; }
.group-title { color: #7f8c8d; font-size: 12px; font-weight: bold; margin: 15px 20px 5px; text-transform: uppercase; letter-spacing: 1px; }

.menu-item { display: block; padding: 12px 20px; color: #ecf0f1; text-decoration: none; font-size: 15px; transition: all 0.3s ease; border-left: 4px solid transparent; }
.menu-item:hover { background: #34495e; color: #fff; padding-left: 25px; }
.router-link-active, .router-link-exact-active { background: #34495e; border-left: 4px solid #f39c12; color: #f39c12; font-weight: bold; }

.pos-link { color: #3498db; }

/* Khu vực Footer chứa nút Profile và Logout */
.sidebar-footer { padding: 15px 20px; background: #1a252f; border-top: 1px solid #34495e; display: flex; flex-direction: column; gap: 10px;}

.profile-btn { display: block; text-align: center; background: #ecf0f1; color: #2c3e50; padding: 10px; border-radius: 5px; text-decoration: none; font-weight: bold; transition: 0.3s; }
.profile-btn:hover { background: #bdc3c7; }

.logout-btn { background: #e74c3c; color: white; border: none; padding: 10px; border-radius: 5px; font-weight: bold; cursor: pointer; transition: 0.3s; font-size: 15px; width: 100%;}
.logout-btn:hover { background: #c0392b; }

/* MAIN CONTENT STYLE */
.main-content { flex: 1; padding: 25px; overflow-y: auto; position: relative; }
::-webkit-scrollbar { width: 8px; }
::-webkit-scrollbar-track { background: #f1f1f1; }
::-webkit-scrollbar-thumb { background: #bdc3c7; border-radius: 4px; }
::-webkit-scrollbar-thumb:hover { background: #95a5a6; }
</style>