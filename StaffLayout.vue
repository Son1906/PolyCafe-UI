<template>
  <div class="staff-layout">
    <header class="staff-header">
      <div class="brand">
        <h2>🧑‍🍳 Nhân Viên PolyCafe</h2>
      </div>
      
      <nav class="nav-links">
        <router-link to="/staff/pos" class="nav-item">💻 Máy POS</router-link>
        <router-link to="/staff/kds" class="nav-item">🔥 Màn hình Pha chế</router-link>
        
        <!-- ĐÃ THÊM NÚT QUẢN LÝ ĐƠN HÀNG VÀO ĐÂY -->
        <router-link to="/staff/orders" class="nav-item">📦 Quản lý Đơn hàng</router-link>
        
        <div class="user-menu" v-if="currentUser">
          <span class="user-name">👤 Ca trực: {{ currentUser.fullName || 'Nhân viên' }}</span>
          <button @click="handleLogout" class="btn-logout">Đăng xuất</button>
        </div>
      </nav>
    </header>

    <main class="main-content">
      <router-view />
    </main>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue';
import { useRouter } from 'vue-router';

const router = useRouter();
const currentUser = ref(null);

onMounted(() => {
  // Lấy thông tin user từ localStorage để hiển thị tên người đang trực máy
  currentUser.value = JSON.parse(localStorage.getItem('user'));
});

// HÀM XỬ LÝ ĐĂNG XUẤT
const handleLogout = () => {
  if (confirm("Bạn có chắc chắn muốn kết thúc ca làm việc và đăng xuất?")) {
    // Xóa thông tin user khỏi trình duyệt
    localStorage.removeItem('user');
    
    // Đẩy về trang Đăng nhập
    router.push('/login');
  }
};
</script>

<style scoped>
.staff-layout {
  display: flex;
  flex-direction: column;
  min-height: 100vh;
}

/* CSS cho Header xịn xò hơn */
.staff-header {
  background: #2c3e50; /* Màu xanh đen sang trọng */
  padding: 0 30px;
  height: 65px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  box-shadow: 0 4px 10px rgba(0,0,0,0.15);
  position: sticky;
  top: 0;
  z-index: 100;
}

.brand h2 {
  color: #f1c40f; /* Màu vàng nổi bật */
  margin: 0;
  font-size: 22px;
  letter-spacing: 1px;
}

.nav-links {
  display: flex;
  align-items: center;
  gap: 10px;
}

.nav-item {
  color: #ecf0f1;
  text-decoration: none;
  font-size: 15px;
  font-weight: bold;
  padding: 10px 15px;
  border-radius: 6px;
  transition: 0.2s;
}

.nav-item:hover {
  background: #34495e;
  color: #3498db;
}

/* Hiệu ứng khi menu đang được chọn */
.router-link-active {
  background: #2980b9;
  color: white;
}

.user-menu {
  display: flex;
  align-items: center;
  gap: 15px;
  border-left: 2px solid #7f8c8d; /* Đường kẻ vạch ngăn cách */
  padding-left: 20px;
  margin-left: 10px;
}

.user-name {
  color: #bdc3c7;
  font-size: 14px;
  font-weight: bold;
}

.btn-logout {
  background: #e74c3c;
  color: white;
  border: none;
  padding: 8px 15px;
  border-radius: 5px;
  cursor: pointer;
  font-weight: bold;
  transition: 0.2s;
}

.btn-logout:hover {
  background: #c0392b;
}

/* Vùng chứa nội dung (Màu nền xám nhạt để nổi bật giao diện POS bên trong) */
.main-content {
  background: #ecf0f1; 
  flex: 1;
}
</style>