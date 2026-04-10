<template>
  <div class="mgmt-page">
    <div class="header">
      <h2>🧋 Quản Lý Topping (Món Thêm)</h2>
      <button class="btn-primary" @click="openAddModal">+ Thêm Topping</button>
    </div>

    <table class="table">
      <thead>
        <tr>
          <th>ID</th>
          <th>Tên Topping</th>
          <th>Giá (VNĐ)</th>
          <th>Trạng thái</th>
          <th>Thao tác</th>
        </tr>
      </thead>
      <tbody>
        <tr v-if="toppings.length === 0">
          <td colspan="5" class="text-center">Chưa có topping nào trong hệ thống.</td>
        </tr>
        <tr v-for="topping in toppings" :key="topping.toppingId || topping.id">
          <td class="code">#{{ topping.toppingId || topping.id }}</td>
          <td><b>{{ topping.toppingName }}</b></td>
          <td class="price">{{ formatPrice(topping.toppingPrice || topping.price) }}</td>
          <td>
            <span class="status" :class="(topping.toppingActive ?? topping.isActive) ? 'active' : 'inactive'">
              {{ (topping.toppingActive ?? topping.isActive) ? 'Đang bán' : 'Ngừng bán' }}
            </span>
          </td>
          <td>
            <button class="btn-edit" @click="openEditModal(topping)">Sửa</button>
            <button class="btn-delete" @click="deleteTopping(topping.toppingId || topping.id)">Xóa</button>
          </td>
        </tr>
      </tbody>
    </table>

    <div class="modal-overlay" v-if="showModal">
      <div class="modal-content">
        <h3>{{ isEditing ? '✏️ Cập Nhật Topping' : '✨ Thêm Topping Mới' }}</h3>
        
        <form @submit.prevent="handleSubmit" class="form-grid">
          <div class="form-group full-width">
            <label>Tên Topping <span class="required">*</span></label>
            <input type="text" v-model="formData.toppingName" placeholder="Ví dụ: Trân châu đen, Thạch nha đam..." required />
          </div>

          <div class="form-group full-width">
            <label>Giá bán (VNĐ) <span class="required">*</span></label>
            <input type="number" v-model="formData.toppingPrice" min="0" required />
          </div>

          <div class="form-group checkbox-group full-width">
            <label class="toggle-label">
              <input type="checkbox" v-model="formData.toppingActive" />
              <span class="toggle-text">Đang kinh doanh (Hiển thị cho khách chọn)</span>
            </label>
          </div>

          <div class="modal-actions full-width">
            <button type="button" class="btn-cancel" @click="closeModal">Hủy</button>
            <button type="submit" class="btn-save" :disabled="loading">
              {{ loading ? 'Đang lưu...' : '💾 Lưu Topping' }}
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

const toppings = ref([]);

// State cho Modal
const showModal = ref(false);
const isEditing = ref(false);
const currentId = ref(null);
const loading = ref(false);

// ĐÃ SỬA THÀNH toppingPrice Ở ĐÂY
const formData = reactive({
  toppingName: '',
  toppingPrice: 0, 
  toppingActive: true
});

onMounted(() => { 
  fetchToppings(); 
});

const fetchToppings = async () => {
  try {
    const res = await api.get('/toppings');
    toppings.value = res.data.data || [];
  } catch (err) {
    console.error("Lỗi lấy topping", err);
  }
};

// ================= LOGIC MODAL =================
const openAddModal = () => {
  isEditing.value = false;
  currentId.value = null;
  // ĐÃ SỬA THÀNH toppingPrice Ở ĐÂY
  Object.assign(formData, { toppingName: '', toppingPrice: 0, toppingActive: true });
  showModal.value = true;
};

const openEditModal = (topping) => {
  isEditing.value = true;
  currentId.value = topping.toppingId || topping.id;
  // ĐÃ SỬA THÀNH toppingPrice Ở ĐÂY
  Object.assign(formData, { 
    toppingName: topping.toppingName, 
    toppingPrice: topping.toppingPrice || topping.price || 0, 
    toppingActive: topping.toppingActive ?? topping.isActive ?? true
  });
  showModal.value = true;
};

const closeModal = () => { 
  showModal.value = false; 
};

// ================= LƯU & XÓA =================
const handleSubmit = async () => {
  loading.value = true;
  try {
    if (isEditing.value) {
      await api.put(`/toppings/${currentId.value}`, formData);
    } else {
      await api.post('/toppings', formData);
    }
    fetchToppings();
    closeModal();
  } catch (err) {
    alert(err.response?.data?.message || 'Có lỗi xảy ra khi lưu Topping!');
  } finally {
    loading.value = false;
  }
};

const deleteTopping = async (id) => {
  if (confirm("Bạn có chắc chắn muốn xóa topping này không?")) {
    try {
      await api.delete(`/toppings/${id}`);
      fetchToppings();
    } catch (err) {
      // Bắt lỗi khóa ngoại nếu topping này đã được bán
      alert(err.response?.data?.message || "Không thể xóa! Topping này đã từng được bán và lưu trong hóa đơn. Hãy chuyển sang trạng thái 'Ngừng bán'.");
    }
  }
};

const formatPrice = (price) => {
  return new Intl.NumberFormat('vi-VN', { style: 'currency', currency: 'VND' }).format(price || 0);
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
.price { color: #d35400; font-weight: bold; }
.text-center { text-align: center; color: #95a5a6; font-style: italic; }

.status { padding: 6px 12px; border-radius: 20px; font-size: 12px; font-weight: bold; }
.status.active { background: #eafaf1; color: #27ae60; }
.status.inactive { background: #fdedec; color: #e74c3c; }

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
.required { color: #e74c3c; }

.checkbox-group { display: flex; align-items: center; margin-top: 5px; }
.toggle-label { display: flex; align-items: center; cursor: pointer; }
.toggle-label input { width: auto; margin-right: 10px; transform: scale(1.2); }
.toggle-text { font-size: 14px; color: #2c3e50; font-weight: bold; }

.modal-actions { display: flex; justify-content: flex-end; gap: 10px; margin-top: 15px; }
.btn-cancel { background: #95a5a6; color: white; border: none; padding: 10px 15px; border-radius: 5px; cursor: pointer; font-weight: bold;}
.btn-cancel:hover { background: #7f8c8d; }
.btn-save { background: #27ae60; color: white; border: none; padding: 10px 20px; border-radius: 5px; font-weight: bold; cursor: pointer; }
.btn-save:hover { background: #2ecc71; }
</style>