<template>
  <div class="mgmt-page">
    <div class="header">
      <h2>🎟️ Quản Lý Mã Giảm Giá (Voucher)</h2>
      <button class="btn-primary" @click="openAddModal">+ Tạo Voucher Mới</button>
    </div>

    <table class="table">
      <thead>
        <tr>
          <th>Mã Code</th>
          <th>Mức Giảm</th>
          <th>Đơn Tối Thiểu</th>
          <th>Hạn Sử Dụng</th>
          <th>Trạng thái</th>
          <th>Thao tác</th>
        </tr>
      </thead>
      <tbody>
        <tr v-if="vouchers.length === 0">
          <td colspan="6" class="text-center">Chưa có mã giảm giá nào.</td>
        </tr>
        <tr v-for="v in vouchers" :key="v.voucherId || v.id">
          <td><span class="code-badge">{{ v.voucherCode }}</span></td>
          <td class="discount">- {{ formatPrice(v.discountAmount) }}</td>
          <td>{{ formatPrice(v.minOrderValue) }}</td>
          <td>
            <span :class="{'text-danger font-bold': isExpired(v.expirationDate)}">
              {{ formatDate(v.expirationDate) }}
              <span v-if="isExpired(v.expirationDate)">(Đã hết hạn)</span>
            </span>
          </td>
          <td>
            <span class="status" :class="v.voucherActive ? 'active' : 'inactive'">
              {{ v.voucherActive ? 'Đang kích hoạt' : 'Đã tắt' }}
            </span>
          </td>
          <td>
            <button class="btn-edit" @click="openEditModal(v)">Sửa</button>
            <button class="btn-delete" @click="deleteVoucher(v.voucherId || v.id)">Xóa</button>
          </td>
        </tr>
      </tbody>
    </table>

    <div class="modal-overlay" v-if="showModal">
      <div class="modal-content">
        <h3>{{ isEditing ? '✏️ Cập Nhật Mã Giảm Giá' : '✨ Tạo Mã Giảm Giá Mới' }}</h3>
        
        <form @submit.prevent="handleSubmit" class="form-grid">
          <div class="form-group full-width">
            <label>Mã Code (Viết liền không dấu) <span class="required">*</span></label>
            <input type="text" v-model="formData.voucherCode" placeholder="VD: GIAM50K, CHAOHE2026..." style="text-transform: uppercase;" required />
          </div>

          <div class="form-row">
            <div class="form-group flex-1">
              <label>Số tiền giảm (VNĐ) <span class="required">*</span></label>
              <input type="number" min="1000" v-model="formData.discountAmount" required />
            </div>
            <div class="form-group flex-1">
              <label>Đơn tối thiểu (VNĐ) <span class="required">*</span></label>
              <input type="number" min="0" v-model="formData.minOrderValue" required />
            </div>
          </div>

          <div class="form-group full-width">
            <label>Ngày hết hạn <span class="required">*</span></label>
            <input type="date" v-model="formData.expirationDate" required />
          </div>

          <div class="form-group checkbox-group full-width">
            <label class="toggle-label">
              <input type="checkbox" v-model="formData.voucherActive" />
              <span class="toggle-text">Cho phép khách hàng sử dụng mã này</span>
            </label>
          </div>

          <div class="modal-actions full-width">
            <button type="button" class="btn-cancel" @click="closeModal">Hủy</button>
            <button type="submit" class="btn-save" :disabled="loading">
              {{ loading ? 'Đang lưu...' : '💾 Lưu Mã Giảm Giá' }}
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

const vouchers = ref([]);

// State Modal
const showModal = ref(false);
const isEditing = ref(false);
const currentId = ref(null);
const loading = ref(false);

const formData = reactive({
  voucherCode: '',
  discountAmount: 0,
  minOrderValue: 0,
  expirationDate: '',
  voucherActive: true
});

onMounted(() => {
  fetchVouchers();
});

const fetchVouchers = async () => {
  try {
    const res = await api.get('/vouchers');
    vouchers.value = res.data.data || [];
  } catch (err) {
    console.error("Lỗi lấy voucher:", err);
  }
};

// ================= MODAL LOGIC =================
const openAddModal = () => {
  isEditing.value = false;
  currentId.value = null;
  // Gán ngày hết hạn mặc định là ngày mai
  const tomorrow = new Date();
  tomorrow.setDate(tomorrow.getDate() + 1);
  
  Object.assign(formData, { 
    voucherCode: '', 
    discountAmount: 0, 
    minOrderValue: 0, 
    expirationDate: tomorrow.toISOString().split('T')[0], 
    voucherActive: true 
  });
  showModal.value = true;
};

const openEditModal = (v) => {
  isEditing.value = true;
  currentId.value = v.voucherId || v.id;
  Object.assign(formData, { 
    voucherCode: v.voucherCode, 
    discountAmount: v.discountAmount, 
    minOrderValue: v.minOrderValue, 
    // Format lại ngày để bind vào thẻ <input type="date">
    expirationDate: v.expirationDate ? new Date(v.expirationDate).toISOString().split('T')[0] : '',
    voucherActive: v.voucherActive 
  });
  showModal.value = true;
};

const closeModal = () => { showModal.value = false; };

// ================= LƯU & XÓA =================
const handleSubmit = async () => {
  loading.value = true;
  // Ép mã code tự động in hoa trước khi gửi
  const payload = { ...formData, voucherCode: formData.voucherCode.toUpperCase() };

  try {
    if (isEditing.value) {
      await api.put(`/vouchers/${currentId.value}`, payload);
    } else {
      await api.post('/vouchers', payload);
    }
    fetchVouchers();
    closeModal();
  } catch (err) {
    alert(err.response?.data?.message || 'Có lỗi xảy ra khi lưu Voucher!');
  } finally {
    loading.value = false;
  }
};

const deleteVoucher = async (id) => {
  if (confirm("Bạn có chắc chắn muốn xóa mã giảm giá này?")) {
    try {
      await api.delete(`/vouchers/${id}`);
      fetchVouchers();
    } catch (err) {
      alert(err.response?.data?.message || "Không thể xóa! Mã giảm giá này đã từng được khách sử dụng trong hóa đơn. Vui lòng chọn 'Sửa' và tắt Trạng thái thay vì xóa.");
    }
  }
};

// ================= UTILITIES =================
const isExpired = (dateString) => {
  if (!dateString) return false;
  // Set thời gian về 00:00:00 để so sánh công bằng trong cùng 1 ngày
  const today = new Date();
  today.setHours(0, 0, 0, 0);
  const expDate = new Date(dateString);
  return expDate < today;
};

const formatPrice = (price) => {
  return new Intl.NumberFormat('vi-VN', { style: 'currency', currency: 'VND' }).format(price || 0);
};

const formatDate = (dateString) => {
  if (!dateString) return '';
  const [year, month, day] = dateString.split('-');
  return `${day}/${month}/${year}`;
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

.code-badge { background: #2c3e50; color: #f1c40f; padding: 6px 12px; border-radius: 4px; font-weight: bold; letter-spacing: 1.5px; box-shadow: 0 2px 4px rgba(0,0,0,0.1);}
.discount { color: #e74c3c; font-weight: bold; font-size: 16px; }
.text-danger { color: #e74c3c; }
.font-bold { font-weight: bold; }
.text-center { text-align: center; color: #95a5a6; font-style: italic; }

.status { padding: 6px 12px; border-radius: 20px; font-size: 12px; font-weight: bold; }
.status.active { background: #eafaf1; color: #27ae60; }
.status.inactive { background: #fdedec; color: #7f8c8d; }

.btn-edit { background: #f39c12; border: none; color: white; padding: 6px 12px; border-radius: 4px; margin-right: 8px; cursor: pointer; font-weight: bold; transition: 0.2s;}
.btn-edit:hover { background: #e67e22; }
.btn-delete { background: #e74c3c; border: none; color: white; padding: 6px 12px; border-radius: 4px; cursor: pointer; font-weight: bold; transition: 0.2s;}
.btn-delete:hover { background: #c0392b; }

/* MODAL STYLES */
.modal-overlay { position: fixed; top: 0; left: 0; width: 100vw; height: 100vh; background: rgba(0,0,0,0.5); display: flex; justify-content: center; align-items: center; z-index: 1000; }
.modal-content { background: white; padding: 25px; border-radius: 10px; width: 500px; max-width: 90%; box-shadow: 0 10px 30px rgba(0,0,0,0.2);}
.modal-content h3 { margin-top: 0; border-bottom: 1px solid #eee; padding-bottom: 10px; color: #2c3e50;}

.form-grid { display: flex; flex-direction: column; gap: 15px; }
.form-row { display: flex; gap: 15px; }
.flex-1 { flex: 1; }
.form-group label { font-size: 13px; color: #7f8c8d; margin-bottom: 5px; font-weight: bold; display: block;}
.form-group input { width: 100%; padding: 10px; border: 1px solid #bdc3c7; border-radius: 5px; box-sizing: border-box; font-family: inherit;}
.form-group input:focus { outline: none; border-color: #3498db; }
.required { color: #e74c3c; }

.checkbox-group { margin-top: 5px; background: #f8f9fa; padding: 10px; border-radius: 5px;}
.toggle-label { display: flex; align-items: center; cursor: pointer; }
.toggle-label input { width: auto; margin-right: 10px; transform: scale(1.2); }
.toggle-text { font-size: 14px; font-weight: bold; color: #2c3e50;}

.modal-actions { display: flex; justify-content: flex-end; gap: 10px; margin-top: 20px; }
.btn-cancel { background: #95a5a6; color: white; border: none; padding: 10px 15px; border-radius: 5px; cursor: pointer; font-weight: bold;}
.btn-cancel:hover { background: #7f8c8d; }
.btn-save { background: #27ae60; color: white; border: none; padding: 10px 20px; border-radius: 5px; font-weight: bold; cursor: pointer; }
.btn-save:hover { background: #2ecc71; }
</style>