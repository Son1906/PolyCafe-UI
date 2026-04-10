<template>
  <div class="profile-page">
    <div class="profile-card">
      <div class="avatar">👤</div>
      <h2>Hồ Sơ Của Bạn</h2>
      
      <div v-if="user" class="info-list">
        
        <div v-if="!isEditing">
          <div class="info-item">
            <label>Họ và Tên:</label>
            <p>{{ user.fullName || 'Chưa cập nhật' }}</p>
          </div>
          <div class="info-item">
            <label>Email:</label>
            <p>{{ user.email }}</p>
          </div>
          <div class="info-item">
            <label>Số điện thoại:</label>
            <p>{{ user.phone || 'Chưa cập nhật' }}</p>
          </div>
          <div class="info-item">
            <label>Địa chỉ:</label>
            <p>{{ user.address || 'Chưa cập nhật' }}</p>
          </div>
          <div class="info-item">
            <label>Cấp bậc:</label>
            <p class="role-badge">{{ getRoleName(user.role) }}</p>
          </div>
          
          <div class="action-buttons">
            <button class="btn-edit" @click="startEditing">✏️ Sửa Thông Tin</button>
            <button class="btn-warning" @click="showPassModal = true">🔑 Đổi Mật Khẩu</button>
          </div>
          <button class="btn-logout" @click="logout">👋 Đăng Xuất</button>
        </div>

        <form v-else @submit.prevent="updateProfile" class="edit-form">
          <div class="form-group">
            <label>Họ và Tên:</label>
            <input type="text" v-model="editData.fullName" required />
          </div>
          <div class="form-group">
            <label>Số điện thoại:</label>
            <input type="text" v-model="editData.phone" />
          </div>
          <div class="form-group">
            <label>Địa chỉ:</label>
            <input type="text" v-model="editData.address" />
          </div>
          
          <div class="action-buttons">
            <button type="button" class="btn-cancel" @click="isEditing = false">Hủy</button>
            <button type="submit" class="btn-save" :disabled="loading">
              {{ loading ? 'Đang lưu...' : '💾 Lưu Thay Đổi' }}
            </button>
          </div>
        </form>

      </div>
      
      <div v-else class="not-logged-in">
        <p>Bạn chưa đăng nhập!</p>
        <router-link to="/login" class="btn-login">Đi tới trang Đăng nhập</router-link>
      </div>
    </div>

    <div class="modal-overlay" v-if="showPassModal">
      <div class="modal-content">
        <h3>Đổi Mật Khẩu</h3>
        <form @submit.prevent="changePassword" class="form-grid">
          <div class="form-group full-width">
            <label>Mật khẩu hiện tại:</label>
            <input type="password" v-model="passData.oldPassword" required />
          </div>
          <div class="form-group full-width">
            <label>Mật khẩu mới:</label>
            <input type="password" v-model="passData.newPassword" required />
          </div>
          <div class="form-group full-width">
            <label>Xác nhận mật khẩu mới:</label>
            <input type="password" v-model="passData.confirmPassword" required />
          </div>
          
          <p v-if="passMsg.text" :class="passMsg.type">{{ passMsg.text }}</p>

          <div class="modal-actions full-width">
            <button type="button" class="btn-cancel" @click="closePassModal">Hủy</button>
            <button type="submit" class="btn-save" :disabled="loading">Xác nhận đổi</button>
          </div>
        </form>
      </div>
    </div>

  </div>
</template>

<script setup>
import { ref, reactive, onMounted } from 'vue';
import { useRouter } from 'vue-router';
import api from '../services/api';

const user = ref(null);
const router = useRouter();
const loading = ref(false);

// State cho phần Sửa thông tin
const isEditing = ref(false);
const editData = reactive({ fullName: '', phone: '', address: '' });

// State cho phần Đổi mật khẩu
const showPassModal = ref(false);
const passData = reactive({ oldPassword: '', newPassword: '', confirmPassword: '' });
const passMsg = ref({ type: '', text: '' });

onMounted(() => {
  const storedUser = localStorage.getItem('user');
  if (storedUser) {
    user.value = JSON.parse(storedUser);
  }
});

const getRoleName = (role) => {
  if (role === 2) return 'Quản lý (Admin)';
  if (role === 1) return 'Nhân viên';
  return 'Khách hàng';
};

// Khởi động chế độ sửa: Copy dữ liệu hiện tại vào form
const startEditing = () => {
  editData.fullName = user.value.fullName || '';
  editData.phone = user.value.phone || '';
  editData.address = user.value.address || '';
  isEditing.value = true;
};

// GỌI API: CẬP NHẬT THÔNG TIN
const updateProfile = async () => {
  loading.value = true;
  try {
    const res = await api.put(`/users/${user.value.userId}`, editData);
    if (res.data.status === 'success') {
      alert('Cập nhật thông tin thành công!');
      // Cập nhật lại UI và LocalStorage
      user.value = res.data.data;
      localStorage.setItem('user', JSON.stringify(user.value));
      isEditing.value = false;
    } else {
      alert(res.data.message || 'Có lỗi xảy ra!');
    }
  } catch (err) {
    alert(err.response?.data?.message || 'Lỗi kết nối API!');
  } finally {
    loading.value = false;
  }
};

// GỌI API: ĐỔI MẬT KHẨU
const changePassword = async () => {
  if (passData.newPassword !== passData.confirmPassword) {
    passMsg.value = { type: 'error-text', text: 'Mật khẩu xác nhận không khớp!' };
    return;
  }

  loading.value = true;
  passMsg.value = { type: '', text: '' };

  try {
    const res = await api.put(`/users/${user.value.userId}/change-password`, {
      oldPassword: passData.oldPassword,
      newPassword: passData.newPassword
    });

    if (res.data.status === 'success') {
      passMsg.value = { type: 'success-text', text: 'Đổi mật khẩu thành công!' };
      setTimeout(() => closePassModal(), 1500); // Đóng popup sau 1.5s
    } else {
      passMsg.value = { type: 'error-text', text: res.data.message || 'Mật khẩu cũ không đúng!' };
    }
  } catch (err) {
    passMsg.value = { type: 'error-text', text: err.response?.data?.message || 'Lỗi server!' };
  } finally {
    loading.value = false;
  }
};

const closePassModal = () => {
  showPassModal.value = false;
  passData.oldPassword = '';
  passData.newPassword = '';
  passData.confirmPassword = '';
  passMsg.value = { type: '', text: '' };
};

const logout = () => {
  localStorage.removeItem('user');
  router.push('/login');
};
</script>

<style scoped>
.profile-page { display: flex; justify-content: center; padding: 50px 20px; }
.profile-card { background: white; padding: 40px; border-radius: 15px; box-shadow: 0 10px 30px rgba(0,0,0,0.1); width: 100%; max-width: 450px; text-align: center; position: relative;}
.avatar { font-size: 60px; background: #ecf0f1; width: 100px; height: 100px; line-height: 100px; border-radius: 50%; margin: 0 auto 20px; }
.info-list { text-align: left; margin-top: 30px; }
.info-item { margin-bottom: 20px; border-bottom: 1px solid #ecf0f1; padding-bottom: 10px; }
.info-item label { color: #7f8c8d; font-size: 14px; font-weight: bold; }
.info-item p { color: #2c3e50; font-size: 18px; margin: 5px 0 0; font-weight: 500; }
.role-badge { display: inline-block; background: #f39c12; color: white; padding: 4px 12px; border-radius: 20px; font-size: 14px; margin-top: 5px !important; }

/* Các nút hành động */
.action-buttons { display: flex; gap: 10px; margin-top: 25px; }
.action-buttons button { flex: 1; padding: 12px; border: none; border-radius: 8px; font-weight: bold; cursor: pointer; transition: 0.3s;}
.btn-edit { background: #3498db; color: white; }
.btn-edit:hover { background: #2980b9; }
.btn-warning { background: #f39c12; color: white; }
.btn-warning:hover { background: #e67e22; }
.btn-cancel { background: #95a5a6; color: white; }
.btn-save { background: #27ae60; color: white; }

.btn-logout { width: 100%; background: #e74c3c; color: white; border: none; padding: 15px; border-radius: 8px; font-size: 16px; font-weight: bold; cursor: pointer; margin-top: 15px; transition: 0.3s; }
.btn-logout:hover { background: #c0392b; }

/* Form Edit */
.edit-form { text-align: left; }
.form-group { display: flex; flex-direction: column; margin-bottom: 15px; }
.form-group label { font-size: 13px; color: #7f8c8d; margin-bottom: 5px; font-weight: bold; }
.form-group input { padding: 10px; border: 1px solid #bdc3c7; border-radius: 5px; }

/* Modal CSS dùng chung */
.modal-overlay { position: fixed; top: 0; left: 0; width: 100vw; height: 100vh; background: rgba(0,0,0,0.5); display: flex; justify-content: center; align-items: center; z-index: 1000; }
.modal-content { background: white; padding: 25px; border-radius: 10px; width: 400px; max-width: 90%; text-align: left;}
.modal-content h3 { margin-top: 0; border-bottom: 1px solid #eee; padding-bottom: 10px; }
.full-width { width: 100%; }
.modal-actions { display: flex; justify-content: flex-end; gap: 10px; margin-top: 20px; }
.error-text { color: #e74c3c; font-size: 14px; font-weight: bold; margin-top: 10px;}
.success-text { color: #27ae60; font-size: 14px; font-weight: bold; margin-top: 10px;}
</style>