<template>
  <div class="mgmt-page">
    <div class="header">
      <h2>📦 Quản Lý Tồn Kho Nguyên Liệu</h2>
      <div class="header-actions">
        <button class="btn-warning" @click="fetchLowStock">🚨 Cảnh Báo Sắp Hết</button>
        <button class="btn-secondary" @click="fetchIngredients">🔄 Tải Lại Kho</button>
        <button class="btn-primary" @click="openAddModal">+ Thêm Nguyên Liệu Mới</button>
        <button class="btn-success" @click="$router.push('/admin/receipts')">📋 Nhập Hàng</button>
      </div>
    </div>

    <table class="table">
      <thead>
        <tr>
          <th>Mã NL</th>
          <th>Tên Nguyên Liệu</th>
          <th>Đơn vị tính</th>
          <th>Tồn kho hiện tại</th>
          <th>Trạng thái HSD</th>
          <th>Thao tác</th>
        </tr>
      </thead>
      <tbody>
        <tr v-if="ingredients.length === 0">
          <td colspan="6" class="text-center">Kho hiện tại đang trống hoặc không có dữ liệu.</td>
        </tr>
        <tr v-for="item in ingredients" :key="item.iId">
          <td>#{{ item.iId }}</td>
          <td><b>{{ item.iName }}</b></td>
          <td>{{ item.unit }}</td>
          <td :class="{'text-danger': item.quantityInStock < 10}">
            <b style="font-size: 16px;">{{ item.quantityInStock }}</b>
          </td>
          <!-- Cột hiển thị hạn sử dụng móc từ LocalStorage -->
          <td>
            <div v-if="getExpirationDate(item.iId)">
              <span :class="checkExpiringSoon(item.iId) ? 'text-danger font-bold blink' : 'text-green'">
                {{ getExpirationDate(item.iId) }}
              </span>
              <span v-if="checkExpiringSoon(item.iId)" class="warning-badge">Sắp hết hạn</span>
            </div>
            <span v-else class="text-muted">Chưa cập nhật</span>
          </td>
          <td>
            <button class="btn-check" @click="openCheckModal(item)">📋 Kiểm kho</button>
            <button class="btn-edit" @click="openEditModal(item)">Sửa</button>
            <button class="btn-delete" @click="deleteIngredient(item.iId)">Xóa</button>
          </td>
        </tr>
      </tbody>
    </table>

    <!-- MODAL THÊM / SỬA NGUYÊN LIỆU (Cũ) -->
    <div class="modal-overlay" v-if="showModal">
      <div class="modal-content">
        <h3>{{ isEditing ? '✏️ Cập Nhật Nguyên Liệu' : '✨ Thêm Nguyên Liệu Mới' }}</h3>
        
        <form @submit.prevent="handleSubmit" class="form-grid">
          <div class="form-group full-width">
            <label>Tên Nguyên Liệu <span class="required">*</span></label>
            <input type="text" v-model="formData.iName" placeholder="Ví dụ: Cà phê hạt, Đường cát..." required />
          </div>

          <div class="form-row">
            <div class="form-group flex-1">
              <label>Đơn vị tính (Pha chế) <span class="required">*</span></label>
              <input type="text" v-model="formData.unit" placeholder="gram, ml..." required />
            </div>
            <div class="form-group flex-1">
              <label>Tồn kho ban đầu</label>
              <input type="number" step="0.1" v-model="formData.quantityInStock" required :disabled="isEditing" />
            </div>
          </div>

          <p v-if="isEditing" class="help-text">💡 Chỉ sửa tên hoặc đơn vị. Để tăng giảm số lượng, hãy dùng Phiếu Nhập hoặc nút Kiểm kho.</p>

          <div class="modal-actions full-width">
            <button type="button" class="btn-cancel" @click="closeModal">Hủy</button>
            <button type="submit" class="btn-save" :disabled="loading">
              {{ loading ? 'Đang lưu...' : '💾 Lưu Vào Kho' }}
            </button>
          </div>
        </form>
      </div>
    </div>

    <!-- MODAL KIỂM KHO CUỐI NGÀY (MỚI) -->
    <div class="modal-overlay" v-if="showCheckModal">
      <div class="modal-content check-modal">
        <h3>📋 Kiểm Kho: <span class="highlight">{{ checkItem?.iName }}</span></h3>
        <p class="system-stock">Tồn kho trên hệ thống: <b>{{ checkItem?.quantityInStock }} {{ checkItem?.unit }}</b></p>
        
        <div class="calculator-box">
          <p class="calc-title">NHẬP SỐ LƯỢNG ĐẾM TAY THỰC TẾ</p>
          
          <div class="form-row align-end">
            <div class="form-group flex-1">
              <label>Hàng CÒN NGUYÊN (Hộp/Bao)</label>
              <input type="number" min="0" v-model="calcCheck.unopened" placeholder="Đếm được..." />
            </div>
            <span class="math-icon">x</span>
            <div class="form-group flex-1">
              <label>Mỗi hộp chứa ({{ checkItem?.unit }})</label>
              <input type="number" step="0.1" min="1" v-model="calcCheck.unitSize" placeholder="VD: 1000" />
            </div>
          </div>
          
          <div class="form-row align-end" style="margin-top: 15px;">
            <div class="form-group full-width">
              <label>Hàng ĐÃ KHUI DỞ (Còn lại bao nhiêu {{ checkItem?.unit }})</label>
              <input type="number" step="0.1" min="0" v-model="calcCheck.opened" placeholder="Cân/Đong được bao nhiêu nhập vào đây..." />
            </div>
          </div>

          <div class="calc-result">
            👉 Tổng số lượng thực tế: <b>{{ actualStock }} {{ checkItem?.unit }}</b>
            <div :class="stockDiff >= 0 ? 'text-green' : 'text-danger'" style="margin-top: 5px; font-size: 13px;">
              Lệch so với phần mềm: <b>{{ stockDiff > 0 ? '+' : '' }}{{ stockDiff }} {{ checkItem?.unit }}</b>
            </div>
          </div>
        </div>

        <div class="modal-actions full-width">
          <button type="button" class="btn-cancel" @click="showCheckModal = false">Hủy</button>
          <button type="button" class="btn-save" @click="submitInventoryCheck" :disabled="loading">
            {{ loading ? 'Đang xử lý...' : '✅ Chốt Tồn Kho' }}
          </button>
        </div>
      </div>
    </div>

  </div>
</template>

<script setup>
import { ref, reactive, computed, onMounted } from 'vue';
import api from '../../services/api';

const ingredients = ref([]);

// State Modal Thêm/Sửa (Cũ)
const showModal = ref(false);
const isEditing = ref(false);
const currentId = ref(null);
const loading = ref(false);

const formData = reactive({
  iName: '',
  unit: '',
  quantityInStock: 0
});

// State Modal Kiểm Kho (Mới)
const showCheckModal = ref(false);
const checkItem = ref(null);
const calcCheck = reactive({
  unopened: 0,   // Số hàng chưa bóc
  unitSize: 1000, // 1 Hộp bao nhiêu gram/ml
  opened: 0      // Số hàng đã khui còn lại
});

// Tự động tính toán Tồn kho thực tế = (Nguyên * Dung tích) + Khui dở
const actualStock = computed(() => {
  return (calcCheck.unopened * calcCheck.unitSize) + calcCheck.opened;
});

// Tính số chênh lệch
const stockDiff = computed(() => {
  if (!checkItem.value) return 0;
  return actualStock.value - checkItem.value.quantityInStock;
});

onMounted(() => {
  fetchIngredients();
});

const fetchIngredients = async () => {
  try {
    const res = await api.get('/ingredients'); 
    ingredients.value = res.data.data || [];
  } catch (err) {
    console.error("Lỗi lấy dữ liệu kho", err);
  }
};

const fetchLowStock = async () => {
  try {
    const res = await api.get('/ingredients/low-stock');
    ingredients.value = res.data.data || [];
  } catch (err) {
    console.error("Lỗi lấy danh sách cảnh báo:", err);
  }
};

// ================= LOGIC HẠN SỬ DỤNG LẤY TỪ LOCAL STORAGE =================
const getExpirationDate = (iId) => {
  const expDates = JSON.parse(localStorage.getItem('ingredient_exp_dates')) || {};
  if (!expDates[iId]) return null;
  const d = new Date(expDates[iId]);
  return d.toLocaleDateString('vi-VN');
};

const checkExpiringSoon = (iId) => {
  const expDates = JSON.parse(localStorage.getItem('ingredient_exp_dates')) || {};
  if (!expDates[iId]) return false;
  
  const expDate = new Date(expDates[iId]).getTime();
  const today = new Date().getTime();
  const diffDays = Math.ceil((expDate - today) / (1000 * 60 * 60 * 24));
  
  return diffDays <= 30 && diffDays >= 0; 
};

// ================= LOGIC THÊM / SỬA (CŨ) =================
const openAddModal = () => {
  isEditing.value = false;
  currentId.value = null;
  Object.assign(formData, { iName: '', unit: '', quantityInStock: 0 });
  showModal.value = true;
};

const openEditModal = (item) => {
  isEditing.value = true;
  currentId.value = item.iId; 
  Object.assign(formData, { iName: item.iName, unit: item.unit, quantityInStock: item.quantityInStock });
  showModal.value = true;
};

const closeModal = () => { showModal.value = false; };

const handleSubmit = async () => {
  loading.value = true;
  try {
    if (isEditing.value) {
      await api.put(`/ingredients/${currentId.value}`, formData);
    } else {
      await api.post('/ingredients', formData);
    }
    fetchIngredients();
    closeModal();
  } catch (err) {
    alert(err.response?.data?.message || 'Có lỗi xảy ra khi lưu vào kho!');
  } finally {
    loading.value = false;
  }
};

const deleteIngredient = async (id) => {
  if (confirm("Bạn có chắc chắn muốn xóa nguyên liệu này?")) {
    try {
      await api.delete(`/ingredients/${id}`);
      // Xóa date trong local storage nếu có
      let expDates = JSON.parse(localStorage.getItem('ingredient_exp_dates')) || {};
      delete expDates[id];
      localStorage.setItem('ingredient_exp_dates', JSON.stringify(expDates));
      fetchIngredients();
    } catch (err) {
      alert(err.response?.data?.message || 'Không thể xóa nguyên liệu này!');
    }
  }
};

// ================= LOGIC KIỂM KHO CUỐI NGÀY (MỚI) =================
const openCheckModal = (item) => {
  checkItem.value = item;
  calcCheck.unopened = 0;
  calcCheck.unitSize = 1000; // Mặc định gợi ý 1000g / 1000ml
  calcCheck.opened = 0;
  showCheckModal.value = true;
};

const submitInventoryCheck = async () => {
  if (!confirm(`Xác nhận chốt tồn kho thực tế là: ${actualStock.value} ${checkItem.value.unit}? (Dữ liệu cũ sẽ bị ghi đè)`)) return;
  
  loading.value = true;
  try {
    // Gọi API sửa thông tin nguyên liệu để đè số tồn kho mới vào
    const payload = {
      iName: checkItem.value.iName,
      unit: checkItem.value.unit,
      quantityInStock: actualStock.value
    };
    
    await api.put(`/ingredients/${checkItem.value.iId}`, payload);
    
    alert("Chốt kho thành công!");
    showCheckModal.value = false;
    fetchIngredients();
  } catch (err) {
    alert("Có lỗi xảy ra khi chốt kho!");
  } finally {
    loading.value = false;
  }
};
</script>

<style scoped>
/* Toàn bộ style cũ giữ nguyên */
.mgmt-page { background: white; padding: 25px; border-radius: 10px; box-shadow: 0 4px 15px rgba(0,0,0,0.05); }
.header { display: flex; justify-content: space-between; margin-bottom: 20px; align-items: center; border-bottom: 2px solid #f4f6f8; padding-bottom: 15px;}
.header h2 { margin: 0; color: #2c3e50; }

.header-actions { display: flex; gap: 10px; }
.header-actions button { padding: 10px 15px; border: none; border-radius: 5px; cursor: pointer; font-weight: bold; transition: 0.3s; color: white;}
.btn-primary { background: #2980b9; } .btn-primary:hover { background: #3498db; }
.btn-secondary { background: #7f8c8d; } .btn-secondary:hover { background: #95a5a6; }
.btn-warning { background: #f39c12; } .btn-warning:hover { background: #e67e22; }
.btn-success { background: #27ae60; } .btn-success:hover { background: #2ecc71; }

.table { width: 100%; border-collapse: collapse; text-align: left; }
.table th, .table td { padding: 15px; border-bottom: 1px solid #eee; vertical-align: middle; }
.table th { background: #f8f9fa; color: #34495e; font-weight: 600; text-transform: uppercase; font-size: 13px; letter-spacing: 0.5px;}
.table tr:hover { background: #fdfdfd; }
.text-danger { color: #e74c3c; }
.text-green { color: #27ae60; font-weight: bold; }
.text-muted { color: #bdc3c7; font-style: italic; font-size: 12px;}
.font-bold { font-weight: bold; }
.text-center { text-align: center; color: #95a5a6; font-style: italic; }

.blink { animation: blinker 1.5s linear infinite; }
@keyframes blinker { 50% { opacity: 0.2; } }

.warning-badge { background: #fdedec; color: #e74c3c; border: 1px solid #e74c3c; font-size: 11px; padding: 2px 6px; border-radius: 4px; font-weight: bold; margin-left: 8px;}

/* NÚT THAO TÁC */
.btn-check { background: #8e44ad; border: none; color: white; padding: 6px 12px; border-radius: 4px; margin-right: 8px; cursor: pointer; font-weight: bold;}
.btn-check:hover { background: #9b59b6; }
.btn-edit { background: #f39c12; border: none; color: white; padding: 6px 12px; border-radius: 4px; margin-right: 8px; cursor: pointer; font-weight: bold;}
.btn-edit:hover { background: #e67e22; }
.btn-delete { background: #e74c3c; border: none; color: white; padding: 6px 12px; border-radius: 4px; cursor: pointer; font-weight: bold;}
.btn-delete:hover { background: #c0392b; }

.modal-overlay { position: fixed; top: 0; left: 0; width: 100vw; height: 100vh; background: rgba(0,0,0,0.5); display: flex; justify-content: center; align-items: center; z-index: 1000; }
.modal-content { background: white; padding: 25px; border-radius: 10px; width: 450px; max-width: 90%; box-shadow: 0 10px 30px rgba(0,0,0,0.2);}
.modal-content h3 { margin-top: 0; border-bottom: 1px solid #eee; padding-bottom: 10px; color: #2c3e50;}

/* CSS MỚI CHO BẢNG KIỂM KHO */
.check-modal { width: 550px; }
.highlight { color: #8e44ad; }
.system-stock { margin-top: -5px; margin-bottom: 20px; font-size: 14px; color: #34495e;}
.system-stock b { color: #2980b9; font-size: 16px;}
.calculator-box { background: #f8f9fa; padding: 20px; border-radius: 8px; border: 1px dashed #bdc3c7;}
.calc-title { margin: 0 0 15px 0; font-size: 13px; font-weight: bold; color: #e67e22;}
.align-end { align-items: flex-end; }
.math-icon { font-size: 20px; font-weight: bold; color: #7f8c8d; margin-bottom: 10px;}
.calc-result { background: #eafaf1; border-left: 4px solid #27ae60; padding: 15px; margin-top: 20px; font-size: 15px; color: #2c3e50;}
.calc-result b { color: #d35400; font-size: 18px;}

.form-grid { display: flex; flex-direction: column; gap: 15px; }
.form-row { display: flex; gap: 15px; }
.flex-1 { flex: 1; }
.full-width { width: 100%; }
.form-group label { font-size: 13px; color: #7f8c8d; margin-bottom: 5px; font-weight: bold; display: block;}
.form-group input { width: 100%; padding: 10px; border: 1px solid #bdc3c7; border-radius: 5px; box-sizing: border-box; font-family: inherit; outline: none;}
.form-group input:focus { border-color: #3498db; }
.form-group input:disabled { background: #ecf0f1; cursor: not-allowed; }
.required { color: #e74c3c; }
.help-text { font-size: 12px; color: #f39c12; margin: 0; font-style: italic; line-height: 1.4; }

.modal-actions { display: flex; justify-content: flex-end; gap: 10px; margin-top: 20px; }
.btn-cancel { background: #95a5a6; color: white; border: none; padding: 10px 15px; border-radius: 5px; cursor: pointer; font-weight: bold;}
.btn-cancel:hover { background: #7f8c8d; }
.btn-save { background: #27ae60; color: white; border: none; padding: 10px 20px; border-radius: 5px; font-weight: bold; cursor: pointer; }
.btn-save:hover { background: #2ecc71; }
.btn-save:disabled { background: #bdc3c7; cursor: not-allowed; }
</style>