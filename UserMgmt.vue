<template>
  <div class="mgmt-page">
    <div class="header">
      <h2>👥 Quản Lý Tài Khoản Hệ Thống</h2>
      <button class="btn-primary" @click="openAddModal">+ Tạo Tài Khoản Mới</button>
    </div>

    <table class="table">
      <thead>
        <tr>
          <th>ID</th>
          <th>Họ và Tên</th>
          <th>Email</th>
          <th>SĐT</th>
          <th>Vai trò (Role)</th>
          <th>Trạng thái</th>
          <th>Thao tác</th>
        </tr>
      </thead>
      <tbody>
        <tr v-if="users.length === 0">
          <td colspan="7" class="text-center">Đang tải dữ liệu...</td>
        </tr>
        <tr v-for="user in users" :key="user.userId">
          <td class="code">#{{ user.userId }}</td>
          <td><b>{{ user.fullName || 'Chưa cập nhật' }}</b></td>
          <td>{{ user.email }}</td>
          <td>{{ user.phone || 'N/A' }}</td>
          <td>
            <span class="role-badge" :class="getRoleClass(user.role)">
              {{ getRoleName(user.role) }}
            </span>
          </td>
          <td>
            <span class="status" :class="user.userActive ? 'active' : 'locked'">
              {{ user.userActive ? 'Hoạt động' : 'Đã khóa' }}
            </span>
          </td>
          <td>
            <button class="btn-edit" @click="openEditModal(user)">Sửa</button>
            <button 
              class="btn-delete" 
              @click="lockUser(user.userId)"
              :disabled="user.userId === currentUser.userId"
              :title="user.userId === currentUser.userId ? 'Không thể tự khóa chính mình' : 'Khóa tài khoản'"
            >
              {{ user.userActive ? '🔒 Khóa' : '🚫 Đã Khóa' }}
            </button>
          </td>
        </tr>
      </tbody>
    </table>

    <div class="modal-overlay" v-if="showModal">
      <div class="modal-content">
        <h3>{{ isEditing ? '✏️ Cập Nhật Tài Khoản' : '✨ Thêm Tài Khoản Mới' }}</h3>
        
        <form @submit.prevent="handleSubmit" class="form-grid">
          <div class="form-group full-width">
            <label>Họ và Tên <span class="required">*</span></label>
            <input type="text" v-model="formData.fullName" required />
          </div>

          <div class="form-row">
            <div class="form-group flex-2">
              <label>Email (Tài khoản đăng nhập) <span v-if="!isEditing" class="required">*</span></label>
              <input type="email" v-model="formData.email" :disabled="isEditing" :required="!isEditing" />
            </div>
            <div class="form-group flex-1" v-if="!isEditing">
              <label>Mật khẩu <span class="required">*</span></label>
              <input type="password" v-model="formData.password" required />
            </div>
          </div>

          <div class="form-row">
            <div class="form-group flex-1">
              <label>Số điện thoại</label>
              <input type="text" v-model="formData.phone" />
            </div>
            <div class="form-group flex-1">
              <label>Cấp bậc (Quyền)</label>
              <select v-model.number="formData.role" required>
                <option value="0">Khách hàng</option>
                <option value="1">Nhân viên (Staff)</option>
                <option value="2">Quản lý (Admin)</option>
              </select>
            </div>
          </div>

          <div class="form-group full-width">
            <label>Địa chỉ liên hệ</label>
            <input type="text" v-model="formData.address" placeholder="Nhập địa chỉ..." />
          </div>

          <div class="form-group checkbox-group" v-if="isEditing">
            <label class="toggle-label">
              <input type="checkbox" v-model="formData.userActive" />
              <span class="toggle-text" :class="formData.userActive ? 'text-green' : 'text-red'">
                {{ formData.userActive ? 'Tài khoản đang Hoạt động' : 'Tài khoản đang bị Khóa' }}
              </span>
            </label>
          </div>

          <div class="modal-actions full-width">
            <button type="button" class="btn-cancel" @click="closeModal">Hủy</button>
            <button type="submit" class="btn-save" :disabled="loading">
              {{ loading ? 'Đang lưu...' : '💾 Lưu Tài Khoản' }}
            </button>
          </div>
        </form>
      </div>
    </div>

  </div>
</template>

<script setup>
import { ref, reactive, onMounted } from 'vue';
import api from '../../services/api';

const users = ref([]);
const showModal = ref(false);
const isEditing = ref(false);
const currentId = ref(null);
const loading = ref(false);

// Lấy ID Admin đang đăng nhập để tránh việc tự khóa tài khoản của chính mình
const currentUser = JSON.parse(localStorage.getItem('user')) || {};

const formData = reactive({
  fullName: '',
  email: '',
  password: '',
  phone: '',
  address: '',
  role: 1, // Mặc định tạo tài khoản thường là tạo cho Nhân viên
  userActive: true
});

onMounted(() => {
  fetchUsers();
});

const fetchUsers = async () => {
  try {
    const res = await api.get('/users');
    users.value = res.data.data || [];
  } catch (err) {
    console.error("Lỗi lấy danh sách user:", err);
  }
};

const getRoleName = (role) => {
  if (role === 2) return 'Quản lý';
  if (role === 1) return 'Nhân viên';
  return 'Khách hàng';
};

const getRoleClass = (role) => {
  if (role === 2) return 'admin';
  if (role === 1) return 'staff';
  return 'customer';
};

// ================= MODAL LOGIC =================
const openAddModal = () => {
  isEditing.value = false;
  currentId.value = null;
  Object.assign(formData, { fullName: '', email: '', password: '', phone: '', address: '', role: 1, userActive: true });
  showModal.value = true;
};

const openEditModal = (user) => {
  isEditing.value = true;
  currentId.value = user.userId;
  // Lúc sửa không cho đổi mật khẩu ở đây, mật khẩu do user tự đổi ở trang Profile
  Object.assign(formData, { 
    fullName: user.fullName || '', 
    email: user.email, 
    phone: user.phone || '', 
    address: user.address || '', 
    role: user.role, 
    userActive: user.userActive 
  });
  showModal.value = true;
};

const closeModal = () => { showModal.value = false; };

// ================= LƯU & KHÓA =================
const handleSubmit = async () => {
  loading.value = true;
  try {
    if (isEditing.value) {
      // Backend: Sửa user không cần gửi password
      await api.put(`/users/${currentId.value}`, {
        fullName: formData.fullName,
        phone: formData.phone,
        address: formData.address,
        role: formData.role,
        userActive: formData.userActive
      });
    } else {
      // Backend: Tạo mới cần đầy đủ
      await api.post('/users', formData);
    }
    fetchUsers();
    closeModal();
  } catch (err) {
    alert(err.response?.data?.message || 'Có lỗi xảy ra khi lưu tài khoản!');
  } finally {
    loading.value = false;
  }
};

const lockUser = async (id) => {
  if (id === currentUser.userId) return; // Bảo vệ an toàn kép

  if (confirm("Bạn có chắc chắn muốn KHÓA tài khoản này? Người dùng sẽ không thể đăng nhập được nữa.")) {
    try {
      // Gọi API DELETE đã được chế lại thành Khóa ở Backend
      await api.delete(`/users/${id}`);
      fetchUsers();
    } catch (err) {
      alert(err.response?.data?.message || "Không thể khóa tài khoản này.");
    }
  }
};
</script>

<style scoped>
.mgmt-page { background: white; padding: 25px; border-radius: 10px; box-shadow: 0 4px 15px rgba(0,0,0,0.05); }
.header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px; border-bottom: 2px solid #f4f6f8; padding-bottom: 15px;}
.header h2 { margin: 0; color: #2c3e50; }

.btn-primary { background: #2980b9; color: white; border: none; padding: 10px 20px; border-radius: 5px; cursor: pointer; font-weight: bold; transition: 0.3s;}
.btn-primary:hover { background: #3498db; }

.table { width: 100%; border-collapse: collapse; text-align: left; }
.table th, .table td { padding: 15px; border-bottom: 1px solid #eee; vertical-align: middle;}
.table th { background: #f8f9fa; color: #34495e; font-weight: 600; text-transform: uppercase; font-size: 13px; letter-spacing: 0.5px;}
.table tr:hover { background: #fdfdfd; }

.code { font-weight: bold; color: #7f8c8d; }
.text-center { text-align: center; color: #95a5a6; font-style: italic; }

.role-badge { padding: 6px 12px; border-radius: 20px; font-size: 12px; font-weight: bold; color: white; display: inline-block; text-align: center; min-width: 80px;}
.role-badge.admin { background: #e74c3c; box-shadow: 0 2px 5px rgba(231,76,60,0.3); }
.role-badge.staff { background: #2980b9; box-shadow: 0 2px 5px rgba(41,128,185,0.3); }
.role-badge.customer { background: #95a5a6; }

.status { padding: 5px 10px; border-radius: 20px; font-size: 12px; font-weight: bold; display: inline-block;}
.status.active { background: #eafaf1; color: #27ae60; border: 1px solid #27ae60;}
.status.locked { background: #fdedec; color: #e74c3c; border: 1px solid #e74c3c;}

.btn-edit { background: #f39c12; border: none; color: white; padding: 6px 12px; border-radius: 4px; margin-right: 8px; cursor: pointer; font-weight: bold; transition: 0.2s;}
.btn-edit:hover { background: #e67e22; }
.btn-delete { background: #7f8c8d; border: none; color: white; padding: 6px 12px; border-radius: 4px; cursor: pointer; font-weight: bold; transition: 0.2s;}
.btn-delete:hover:not(:disabled) { background: #34495e; }
.btn-delete:disabled { opacity: 0.5; cursor: not-allowed; }

/* MODAL STYLES */
.modal-overlay { position: fixed; top: 0; left: 0; width: 100vw; height: 100vh; background: rgba(0,0,0,0.5); display: flex; justify-content: center; align-items: center; z-index: 1000; }
.modal-content { background: white; padding: 25px; border-radius: 10px; width: 550px; max-width: 95%; box-shadow: 0 10px 30px rgba(0,0,0,0.2);}
.modal-content h3 { margin-top: 0; border-bottom: 1px solid #eee; padding-bottom: 10px; color: #2c3e50;}

.form-grid { display: flex; flex-direction: column; gap: 15px; }
.form-row { display: flex; gap: 15px; }
.flex-1 { flex: 1; } .flex-2 { flex: 2; }
.form-group label { font-size: 13px; color: #7f8c8d; margin-bottom: 5px; font-weight: bold; display: block;}
.form-group input, .form-group select { width: 100%; padding: 10px; border: 1px solid #bdc3c7; border-radius: 5px; box-sizing: border-box; font-family: inherit;}
.form-group input:focus, .form-group select:focus { outline: none; border-color: #3498db; }
.form-group input:disabled { background: #ecf0f1; cursor: not-allowed; color: #7f8c8d; }
.required { color: #e74c3c; }

.checkbox-group { margin-top: 5px; background: #f8f9fa; padding: 10px; border-radius: 5px;}
.toggle-label { display: flex; align-items: center; cursor: pointer; }
.toggle-label input { width: auto; margin-right: 10px; transform: scale(1.2); }
.toggle-text { font-size: 14px; font-weight: bold; }
.text-green { color: #27ae60; }
.text-red { color: #e74c3c; }

.modal-actions { display: flex; justify-content: flex-end; gap: 10px; margin-top: 20px; }
.btn-cancel { background: #95a5a6; color: white; border: none; padding: 10px 15px; border-radius: 5px; cursor: pointer; font-weight: bold;}
.btn-cancel:hover { background: #7f8c8d; }
.btn-save { background: #27ae60; color: white; border: none; padding: 10px 20px; border-radius: 5px; font-weight: bold; cursor: pointer; }
.btn-save:hover { background: #2ecc71; }
</style>