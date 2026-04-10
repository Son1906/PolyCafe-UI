<template>
  <div class="mgmt-page">
    <div class="header">
      <h2>📁 Quản Lý Danh Mục Đồ Uống</h2>
      <button class="btn-primary" @click="openAddModal">+ Thêm Danh Mục</button>
    </div>

    <table class="table">
      <thead>
        <tr>
          <th>ID</th>
          <th>Tên Danh Mục</th>
          <th>Thao tác</th>
        </tr>
      </thead>
      <tbody>
        <tr v-if="categories.length === 0">
          <td colspan="3" class="text-center">Chưa có danh mục nào.</td>
        </tr>
        <tr v-for="cat in categories" :key="cat.categoryId">
          <td class="code">#{{ cat.categoryId }}</td>
          <td><b>{{ cat.categoryName }}</b></td>
          <td>
            <button class="btn-edit" @click="openEditModal(cat)">Sửa</button>
            <button class="btn-delete" @click="deleteCat(cat.categoryId)">Xóa</button>
          </td>
        </tr>
      </tbody>
    </table>

    <div class="modal-overlay" v-if="showModal">
      <div class="modal-content">
        <h3>{{ isEditing ? '✏️ Sửa Danh Mục' : '✨ Thêm Danh Mục Mới' }}</h3>
        
        <form @submit.prevent="handleSubmit" class="form-grid">
          <div class="form-group full-width">
            <label>Tên danh mục:</label>
            <input type="text" v-model="formData.categoryName" placeholder="Ví dụ: Cà phê pha máy, Trà trái cây..." required />
          </div>

          <div class="modal-actions full-width">
            <button type="button" class="btn-cancel" @click="closeModal">Hủy</button>
            <button type="submit" class="btn-save" :disabled="loading">
              {{ loading ? 'Đang lưu...' : 'Lưu Danh Mục' }}
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

const categories = ref([]);

// State cho Modal
const showModal = ref(false);
const isEditing = ref(false);
const currentId = ref(null);
const loading = ref(false);

const formData = reactive({
  categoryName: ''
});

onMounted(() => {
  fetchCategories();
});

const fetchCategories = async () => {
  try {
    const res = await api.get('/categories'); 
    categories.value = res.data.data || [];
  } catch (err) {
    console.error("Lỗi lấy danh mục", err);
  }
};

// ================= LOGIC MODAL =================
const openAddModal = () => {
  isEditing.value = false;
  currentId.value = null;
  formData.categoryName = '';
  showModal.value = true;
};

const openEditModal = (cat) => {
  isEditing.value = true;
  currentId.value = cat.categoryId;
  formData.categoryName = cat.categoryName;
  showModal.value = true;
};

const closeModal = () => {
  showModal.value = false;
  formData.categoryName = '';
};

// ================= LOGIC LƯU (THÊM/SỬA) =================
const handleSubmit = async () => {
  loading.value = true;
  try {
    if (isEditing.value) {
      // Gọi API Sửa (PUT)
      await api.put(`/categories/${currentId.value}`, formData);
    } else {
      // Gọi API Thêm (POST)
      await api.post('/categories', formData);
    }
    fetchCategories(); // Tải lại bảng
    closeModal();      // Đóng popup
  } catch (err) {
    alert(err.response?.data?.message || 'Có lỗi xảy ra khi lưu!');
  } finally {
    loading.value = false;
  }
};

// ================= LOGIC XÓA =================
const deleteCat = async (id) => {
  if (confirm("Bạn có chắc chắn muốn xóa danh mục này?")) {
    try {
      const res = await api.delete(`/categories/${id}`);
      if (res.data.status === 'success') {
        fetchCategories();
      } else {
        alert(res.data.message); // Backend trả về thông báo lỗi khóa ngoại
      }
    } catch (err) {
      // Bắt lỗi khi Backend quăng Exception chặn xóa (Có chứa món bên trong)
      alert(err.response?.data?.message || 'Không thể xóa danh mục này! Hãy kiểm tra xem có đồ uống nào đang thuộc danh mục này không.');
    }
  }
};
</script>

<style scoped>
.mgmt-page { background: white; padding: 25px; border-radius: 10px; box-shadow: 0 4px 15px rgba(0,0,0,0.05); }
.header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px; border-bottom: 2px solid #f4f6f8; padding-bottom: 15px;}
.header h2 { margin: 0; color: #2c3e50; }

.btn-primary { background: #2980b9; color: white; padding: 10px 20px; border: none; border-radius: 5px; cursor: pointer; font-weight: bold; transition: 0.3s;}
.btn-primary:hover { background: #3498db; }

.table { width: 100%; border-collapse: collapse; text-align: left; }
.table th, .table td { padding: 15px; border-bottom: 1px solid #eee; vertical-align: middle;}
.table th { background: #f8f9fa; color: #34495e; font-weight: 600; text-transform: uppercase; font-size: 13px; letter-spacing: 0.5px;}
.table tr:hover { background: #fdfdfd; }
.code { font-weight: bold; color: #8e44ad; }
.text-center { text-align: center; color: #95a5a6; font-style: italic; }

.btn-edit { background: #f39c12; border: none; color: white; padding: 6px 12px; border-radius: 4px; margin-right: 8px; cursor: pointer; font-weight: bold; transition: 0.2s;}
.btn-edit:hover { background: #e67e22; }
.btn-delete { background: #e74c3c; border: none; color: white; padding: 6px 12px; border-radius: 4px; cursor: pointer; font-weight: bold; transition: 0.2s;}
.btn-delete:hover { background: #c0392b; }

/* MODAL STYLES */
.modal-overlay { position: fixed; top: 0; left: 0; width: 100vw; height: 100vh; background: rgba(0,0,0,0.5); display: flex; justify-content: center; align-items: center; z-index: 1000; }
.modal-content { background: white; padding: 25px; border-radius: 10px; width: 400px; max-width: 90%; box-shadow: 0 10px 30px rgba(0,0,0,0.2);}
.modal-content h3 { margin-top: 0; border-bottom: 1px solid #eee; padding-bottom: 10px; color: #2c3e50;}

.form-grid { display: flex; flex-direction: column; gap: 15px; }
.form-group label { font-size: 13px; color: #7f8c8d; margin-bottom: 5px; font-weight: bold; display: block;}
.form-group input { width: 100%; padding: 10px; border: 1px solid #bdc3c7; border-radius: 5px; box-sizing: border-box; font-family: inherit;}
.form-group input:focus { outline: none; border-color: #3498db; }

.modal-actions { display: flex; justify-content: flex-end; gap: 10px; margin-top: 15px; }
.btn-cancel { background: #95a5a6; color: white; border: none; padding: 10px 15px; border-radius: 5px; cursor: pointer; font-weight: bold;}
.btn-cancel:hover { background: #7f8c8d; }
.btn-save { background: #27ae60; color: white; border: none; padding: 10px 20px; border-radius: 5px; font-weight: bold; cursor: pointer; }
.btn-save:hover { background: #2ecc71; }
</style>