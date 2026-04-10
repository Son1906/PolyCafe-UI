<template>
  <div class="login-wrapper">
    <div class="login-card">
      <h1 class="logo">☕ PolyCafe</h1>
      <p class="subtitle">{{ isLoginMode ? 'Đăng nhập hệ thống' : 'Đăng ký tài khoản mới' }}</p>
      
      <form v-if="isLoginMode" @submit.prevent="handleLogin" class="form">
        <input type="email" v-model="email" placeholder="Nhập Email của bạn" required />
        <input type="password" v-model="password" placeholder="Nhập mật khẩu" required />
        
        <button type="submit" class="btn-submit" :disabled="loading">
          {{ loading ? 'Đang xử lý...' : 'ĐĂNG NHẬP' }}
        </button>

        <p class="toggle-text">
          Chưa có tài khoản? 
          <span class="link" @click="toggleMode">Đăng ký ngay</span>
        </p>
      </form>

      <form v-else @submit.prevent="handleRegister" class="form">
        <input type="text" v-model="regFullName" placeholder="Họ và tên của bạn" required />
        <input type="text" v-model="regPhone" placeholder="Số điện thoại" required />
        <input type="email" v-model="email" placeholder="Nhập Email" required />
        <input type="password" v-model="password" placeholder="Tạo mật khẩu" required />
        
        <button type="submit" class="btn-submit reg-btn" :disabled="loading">
          {{ loading ? 'Đang xử lý...' : 'ĐĂNG KÝ TÀI KHOẢN' }}
        </button>

        <p class="toggle-text">
          Đã có tài khoản? 
          <span class="link" @click="toggleMode">Đăng nhập</span>
        </p>
      </form>

      <p v-if="errorMsg" class="error">{{ errorMsg }}</p>
      <p v-if="successMsg" class="success">{{ successMsg }}</p>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue';
import { useRouter } from 'vue-router';
import api from '../services/api';

const router = useRouter();

// Trạng thái Form (true = Đăng nhập, false = Đăng ký)
const isLoginMode = ref(true);

// Biến lưu dữ liệu chung
const email = ref('');
const password = ref('');
const regFullName = ref('');
const regPhone = ref('');

// Biến thông báo & loading
const errorMsg = ref('');
const successMsg = ref('');
const loading = ref(false);

// Hàm chuyển đổi giữa 2 form
const toggleMode = () => {
  isLoginMode.value = !isLoginMode.value;
  errorMsg.value = '';
  successMsg.value = '';
  password.value = ''; // Xóa password cho an toàn khi lật form
};

// ================= LOGIC ĐĂNG NHẬP =================
const handleLogin = async () => {
  loading.value = true;
  errorMsg.value = '';
  successMsg.value = '';
  localStorage.removeItem('user'); // Dọn rác phiên cũ

  try {
    const res = await api.post('/users/login', { 
      email: email.value, 
      password: password.value 
    });

    if (res.data.status === 'success') {
      const user = res.data.data;
      localStorage.setItem('user', JSON.stringify(user));

      // Phân quyền chuyển hướng
      if (user.role == 2) {
        router.push('/admin'); // Admin
      } else if (user.role == 1) {
        router.push('/staff'); // Nhân viên
      } else {
        router.push('/');      // Khách hàng
      }
    } else {
      errorMsg.value = res.data.message || 'Sai tài khoản hoặc mật khẩu!';
    }
  } catch (err) {
    errorMsg.value = err.response?.data?.message || "Lỗi kết nối máy chủ!";
  } finally {
    loading.value = false;
  }
};

// ================= LOGIC ĐĂNG KÝ =================
const handleRegister = async () => {
  loading.value = true;
  errorMsg.value = '';
  successMsg.value = '';

  try {
    const payload = {
      fullName: regFullName.value,
      phone: regPhone.value,
      email: email.value,
      password: password.value
      // Role và Trạng thái (Active) đã được Backend tự động set
    };

    const res = await api.post('/users/register', payload);

    if (res.data.status === 'success') {
      successMsg.value = 'Đăng ký thành công! Vui lòng đăng nhập.';
      // Sau khi đăng ký xong thì đẩy về lại màn hình Đăng nhập
      isLoginMode.value = true; 
      password.value = ''; 
    } else {
      errorMsg.value = res.data.message || 'Đăng ký thất bại!';
    }
  } catch (err) {
    // Bắt lỗi trùng Email từ Backend trả về
    errorMsg.value = err.response?.data?.message || "Lỗi máy chủ! Không thể đăng ký.";
  } finally {
    loading.value = false;
  }
};
</script>

<style scoped>
.login-wrapper { height: 100vh; display: flex; align-items: center; justify-content: center; background: #8B4513; }
.login-card { background: white; padding: 40px; border-radius: 10px; width: 350px; text-align: center; box-shadow: 0 4px 10px rgba(0,0,0,0.2); }
.logo { color: #8B4513; margin: 0; font-size: 32px; }
.subtitle { color: #7f8c8d; margin-bottom: 25px; }
.form input { width: 100%; padding: 12px; margin-bottom: 15px; border: 1px solid #ccc; border-radius: 5px; box-sizing: border-box; }

.btn-submit { width: 100%; padding: 12px; background: #d35400; color: white; border: none; border-radius: 5px; font-weight: bold; cursor: pointer; transition: 0.3s; }
.btn-submit:hover { background: #e67e22; }
.reg-btn { background: #27ae60; }
.reg-btn:hover { background: #2ecc71; }

.toggle-text { margin-top: 20px; font-size: 14px; color: #7f8c8d; }
.link { color: #2980b9; font-weight: bold; cursor: pointer; text-decoration: underline; }
.link:hover { color: #3498db; }

.error { color: #e74c3c; margin-top: 15px; font-weight: bold; font-size: 14px;}
.success { color: #27ae60; margin-top: 15px; font-weight: bold; font-size: 14px;}
</style>