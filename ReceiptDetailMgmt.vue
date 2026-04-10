<template>
  <div class="mgmt-page">
    <div class="header">
      <div class="header-left">
        <button class="btn-back" @click="$router.push('/admin/receipts')">🔙 Quay Lại</button>
        <h2>📋 Chi Tiết Phiếu Nhập <span class="highlight">#{{ receiptId }}</span></h2>
      </div>
      <div>
        <button class="btn-primary" @click="openAddModal">+ Thêm Nguyên Liệu</button>
      </div>
    </div>

    <div class="summary-box">
      <span>Tổng giá trị phiếu nhập hiện tại:</span>
      <b class="total-price">{{ formatPrice(receiptTotal) }}</b>
    </div>

    <table class="table">
      <thead>
        <tr>
          <th>Mã NL</th>
          <th>Tên Nguyên Liệu</th>
          <th>Thực Nhập (Quy đổi)</th>
          <th>Hạn Sử Dụng</th>
          <th>Đơn Giá (Theo đơn vị gốc)</th>
          <th>Thành Tiền</th>
          <th>Thao tác</th>
        </tr>
      </thead>
      <tbody>
        <tr v-if="details.length === 0">
          <td colspan="7" class="text-center">Phiếu nhập này chưa có nguyên liệu nào. Hãy bấm thêm mới!</td>
        </tr>
        <tr v-for="d in details" :key="d.irdId">
          <td class="code">#{{ d.ingredient?.iId }}</td>
          <td><b>{{ d.ingredient?.iName }}</b></td>
          <td>
            <span class="qty-badge">{{ d.irdQuantity }} {{ d.ingredient?.unit }}</span>
          </td>
          <!-- Lấy hạn sử dụng từ LocalStorage -->
          <td :class="{'text-danger': checkExpiringSoon(d.ingredient?.iId)}">
            <b>{{ getExpirationDate(d.ingredient?.iId) || 'Không có' }}</b>
            <span v-if="checkExpiringSoon(d.ingredient?.iId)" style="font-size: 11px; display: block;">(Sắp hết hạn!)</span>
          </td>
          <td>{{ formatPrice(d.unitPrice) }} / {{ d.ingredient?.unit }}</td>
          <td class="price">{{ formatPrice(d.irdQuantity * d.unitPrice) }}</td>
          <td>
            <button class="btn-delete" @click="deleteDetail(d.irdId, d.ingredient?.iId)">🗑️ Xóa</button>
          </td>
        </tr>
      </tbody>
    </table>

    <div class="modal-overlay" v-if="showModal">
      <div class="modal-content">
        <h3>✨ Thêm Nguyên Liệu Vào Phiếu</h3>
        
        <form @submit.prevent="saveDetail" class="form-grid">
          <div class="form-group full-width">
            <label>Chọn Nguyên Liệu từ Kho: <span class="required">*</span></label>
            <select v-model="formData.ingredient.iId" @change="updateSelectedUnit" required>
              <option value="" disabled>-- Bấm để chọn nguyên liệu --</option>
              <option v-for="ing in ingredients" :key="ing.iId" :value="ing.iId">
                {{ ing.iName }} (Tồn hiện tại: {{ ing.quantityInStock }} {{ ing.unit }})
              </option>
            </select>
          </div>

          <!-- BỘ TÍNH TOÁN QUY ĐỔI THÔNG MINH (Không cần lưu DB) -->
          <div class="calculator-box">
            <p class="calc-title">TÍNH TOÁN QUY ĐỔI NHẬP HÀNG</p>
            <div class="form-row">
              <div class="form-group flex-1">
                <label>Nhập theo (VD: Thùng, Bao)</label>
                <input type="text" v-model="calc.importUnit" placeholder="Thùng..." />
              </div>
              <div class="form-group flex-1">
                <label>Số lượng nhập <span class="required">*</span></label>
                <input type="number" step="0.1" min="0.1" v-model="calc.quantity" required />
              </div>
            </div>
            
            <div class="form-row">
              <div class="form-group flex-1">
                <label>Quy đổi ra {{ selectedUnit || 'Đơn vị gốc' }} <span class="required">*</span></label>
                <input type="number" step="0.1" min="1" v-model="calc.conversionRate" placeholder="1 Thùng = ... ml/gr" required />
              </div>
              <div class="form-group flex-1">
                <label>Tổng tiền thanh toán (VNĐ)</label>
                <input type="number" min="0" v-model="calc.totalPay" placeholder="VNĐ" required />
              </div>
            </div>
            
            <div class="calc-result" v-if="calc.quantity > 0 && calc.conversionRate > 0">
              👉 Sẽ cộng vào kho: <b>{{ finalCalculatedQty }} {{ selectedUnit }}</b> 
              <br>
              👉 Đơn giá: <b>{{ formatPrice(calculatedUnitPrice) }} / {{ selectedUnit }}</b>
            </div>
          </div>

          <!-- HẠN SỬ DỤNG LƯU LOCALSTORAGE -->
          <div class="form-group full-width">
            <label>Hạn sử dụng của lô này (Cảnh báo 1 tháng)</label>
            <input type="date" v-model="formData.expirationDate" />
          </div>

          <div class="modal-actions full-width">
            <button type="button" class="btn-cancel" @click="showModal = false">Hủy</button>
            <button type="submit" class="btn-save" :disabled="loading || !formData.ingredient.iId">
              {{ loading ? 'Đang xử lý...' : '💾 Lưu & Cộng Kho' }}
            </button>
          </div>
        </form>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, reactive, onMounted, computed } from 'vue';
import { useRoute, useRouter } from 'vue-router';
import api from '../../services/api';

const route = useRoute();
const router = useRouter();
const receiptId = route.params.id; 

const details = ref([]);
const ingredients = ref([]);
const showModal = ref(false);
const loading = ref(false);
const selectedUnit = ref('');

// Object phụ trợ để tính toán quy đổi
const calc = reactive({
  importUnit: 'Thùng',
  quantity: 1,
  conversionRate: 1,
  totalPay: 0
});

const receiptTotal = computed(() => {
  return details.value.reduce((sum, item) => sum + (item.irdQuantity * item.unitPrice), 0);
});

// Kết quả quy đổi tự động đẩy vào formData
const finalCalculatedQty = computed(() => calc.quantity * calc.conversionRate);
const calculatedUnitPrice = computed(() => {
  if (finalCalculatedQty.value === 0) return 0;
  return Math.round(calc.totalPay / finalCalculatedQty.value);
});

const formData = reactive({
  irdQuantity: 0,
  unitPrice: 0,
  inventoryReceipt: { irId: receiptId },
  ingredient: { iId: '' },
  expirationDate: '' // Biến này không có trong DB, sẽ lưu xuống localStorage
});

onMounted(async () => {
  fetchDetails();
  try {
    const resIng = await api.get('/ingredients');
    ingredients.value = resIng.data.data || [];
  } catch (err) {
    console.error("Lỗi tải nguyên liệu:", err);
  }
});

const fetchDetails = async () => {
  try {
    const res = await api.get(`/receipt-details/receipt/${receiptId}`);
    details.value = res.data.data || [];
  } catch (err) {
    console.error("Lỗi lấy chi tiết phiếu nhập:", err);
  }
};

const syncReceiptTotal = async () => {
  try {
    let calculatedTotal = 0;
    details.value.forEach(item => { calculatedTotal += (item.irdQuantity * item.unitPrice); });
    const resReceipt = await api.get(`/receipts/${receiptId}`);
    const currentReceipt = resReceipt.data.data;
    currentReceipt.totalCost = Math.round(calculatedTotal);
    await api.put(`/receipts/${receiptId}`, currentReceipt);
  } catch (err) {
    console.error("Lỗi đồng bộ tổng tiền:", err);
  }
};

const updateSelectedUnit = () => {
  const ing = ingredients.value.find(i => i.iId === formData.ingredient.iId);
  selectedUnit.value = ing ? ing.unit : 'Đơn vị gốc';
};

const openAddModal = () => {
  Object.assign(calc, { importUnit: 'Thùng', quantity: 1, conversionRate: 1, totalPay: 0 });
  Object.assign(formData, { irdQuantity: 0, unitPrice: 0, inventoryReceipt: { irId: receiptId }, ingredient: { iId: '' }, expirationDate: '' });
  selectedUnit.value = '';
  showModal.value = true;
};

const saveDetail = async () => {
  loading.value = true;
  // Gán kết quả quy đổi vào form data gửi API
  formData.irdQuantity = finalCalculatedQty.value;
  formData.unitPrice = calculatedUnitPrice.value;

  try {
    // 1. Lưu DB phiếu nhập (Backend của bạn chỉ nhận irdQuantity và unitPrice)
    const payload = { ...formData };
    delete payload.expirationDate; // Xóa trước khi gửi để tránh lỗi DB
    await api.post('/receipt-details', payload);
    
    // 2. LƯU HẠN SỬ DỤNG VÀO LOCALSTORAGE (Vì DB không cho đổi)
    if (formData.expirationDate) {
      let expDates = JSON.parse(localStorage.getItem('ingredient_exp_dates')) || {};
      expDates[formData.ingredient.iId] = formData.expirationDate;
      localStorage.setItem('ingredient_exp_dates', JSON.stringify(expDates));
    }

    showModal.value = false;
    await fetchDetails(); 
    await syncReceiptTotal(); 
    
    const resIng = await api.get('/ingredients');
    ingredients.value = resIng.data.data || [];
    
  } catch (err) {
    alert('Lỗi: ' + (err.response?.data?.message || 'Không thể lưu.'));
  } finally {
    loading.value = false;
  }
};

const deleteDetail = async (id, iId) => {
  if (confirm("Xóa dòng nhập này? Số lượng trong kho sẽ tự động bị trừ đi tương ứng.")) {
    try {
      await api.delete(`/receipt-details/${id}`);
      
      // Xóa luôn HSD trong localStorage nếu cần
      let expDates = JSON.parse(localStorage.getItem('ingredient_exp_dates')) || {};
      delete expDates[iId];
      localStorage.setItem('ingredient_exp_dates', JSON.stringify(expDates));

      await fetchDetails(); 
      await syncReceiptTotal(); 
    } catch (err) {
      alert("Lỗi khi xóa: " + (err.response?.data?.message || ""));
    }
  }
};

// ĐỌC HẠN SỬ DỤNG TỪ LOCALSTORAGE
const getExpirationDate = (iId) => {
  const expDates = JSON.parse(localStorage.getItem('ingredient_exp_dates')) || {};
  if (!expDates[iId]) return null;
  const d = new Date(expDates[iId]);
  return d.toLocaleDateString('vi-VN');
};

// CẢNH BÁO 1 THÁNG (30 NGÀY)
const checkExpiringSoon = (iId) => {
  const expDates = JSON.parse(localStorage.getItem('ingredient_exp_dates')) || {};
  if (!expDates[iId]) return false;
  
  const expDate = new Date(expDates[iId]).getTime();
  const today = new Date().getTime();
  const diffDays = Math.ceil((expDate - today) / (1000 * 60 * 60 * 24));
  
  return diffDays <= 30 && diffDays >= 0; // Còn dưới 30 ngày
};

const formatPrice = (p) => new Intl.NumberFormat('vi-VN', { style: 'currency', currency: 'VND' }).format(p || 0);
</script>

<style scoped>
/* Toàn bộ style cũ giữ nguyên */
.mgmt-page { background: white; padding: 25px; border-radius: 10px; box-shadow: 0 4px 15px rgba(0,0,0,0.05); }
.header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px; border-bottom: 2px solid #f4f6f8; padding-bottom: 15px;}
.header-left { display: flex; align-items: center; gap: 15px; }
.header h2 { margin: 0; color: #2c3e50; }
.highlight { color: #8e44ad; }

.summary-box { background: #fdf2e9; border-left: 5px solid #e67e22; padding: 15px 20px; border-radius: 5px; margin-bottom: 20px; display: flex; justify-content: space-between; align-items: center;}
.summary-box span { font-size: 15px; color: #d35400; font-weight: bold; }
.total-price { font-size: 24px; color: #c0392b; }

.btn-primary { background: #2980b9; color: white; padding: 10px 20px; border: none; border-radius: 5px; cursor: pointer; font-weight: bold; transition: 0.3s;}
.btn-primary:hover { background: #3498db; }
.btn-back { background: #7f8c8d; color: white; padding: 10px 15px; border: none; border-radius: 5px; cursor: pointer; font-weight: bold; transition: 0.3s;}
.btn-back:hover { background: #95a5a6; }

.table { width: 100%; border-collapse: collapse; text-align: left; }
.table th, .table td { padding: 15px; border-bottom: 1px solid #eee; vertical-align: middle;}
.table th { background: #f8f9fa; color: #34495e; font-weight: 600; text-transform: uppercase; font-size: 13px; letter-spacing: 0.5px;}
.table tr:hover { background: #fdfdfd; }
.text-danger { color: #e74c3c; animation: pulse 1.5s infinite; }
@keyframes pulse { 0% { opacity: 1; } 50% { opacity: 0.5; } 100% { opacity: 1; } }

.code { font-weight: bold; color: #7f8c8d; }
.price { color: #d35400; font-weight: bold; }
.qty-badge { background: #e8f4f8; color: #2980b9; padding: 4px 10px; border-radius: 4px; font-size: 13px; font-weight: bold;}
.text-center { text-align: center; color: #95a5a6; font-style: italic; }

.btn-delete { background: #e74c3c; color: white; border: none; padding: 6px 12px; border-radius: 4px; cursor: pointer; font-weight: bold; transition: 0.2s;}
.btn-delete:hover { background: #c0392b; }

.modal-overlay { position: fixed; top: 0; left: 0; width: 100vw; height: 100vh; background: rgba(0,0,0,0.5); display: flex; justify-content: center; align-items: center; z-index: 1000; }
.modal-content { background: white; padding: 25px; border-radius: 10px; width: 550px; max-width: 90%; box-shadow: 0 10px 30px rgba(0,0,0,0.2); }
.modal-content h3 { margin-top: 0; border-bottom: 1px solid #eee; padding-bottom: 10px; color: #2c3e50;}

/* CSS MỚI CHO BỘ TÍNH TOÁN QUY ĐỔI */
.calculator-box { background: #f8f9fa; padding: 15px; border-radius: 8px; border: 1px dashed #bdc3c7; margin-bottom: 15px;}
.calc-title { margin: 0 0 10px 0; font-size: 12px; font-weight: bold; color: #2980b9; text-transform: uppercase;}
.calc-result { background: #eafaf1; border-left: 4px solid #27ae60; padding: 10px; margin-top: 10px; font-size: 14px; color: #2c3e50;}

.form-grid { display: flex; flex-direction: column; gap: 15px; }
.form-row { display: flex; gap: 15px; margin-bottom: 10px;}
.flex-1 { flex: 1; } .flex-2 { flex: 2; }
.form-group label { font-size: 13px; color: #7f8c8d; margin-bottom: 5px; font-weight: bold; display: block;}
.form-group input, .form-group select { width: 100%; padding: 10px; border: 1px solid #bdc3c7; border-radius: 5px; box-sizing: border-box; font-family: inherit;}
.form-group input:focus, .form-group select:focus { outline: none; border-color: #3498db; }
.required { color: #e74c3c; }

.modal-actions { display: flex; justify-content: flex-end; gap: 10px; margin-top: 15px; }
.btn-cancel { background: #95a5a6; color: white; border: none; padding: 10px 15px; border-radius: 5px; cursor: pointer; font-weight: bold;}
.btn-cancel:hover { background: #7f8c8d; }
.btn-save { background: #27ae60; color: white; border: none; padding: 10px 20px; border-radius: 5px; font-weight: bold; cursor: pointer; }
.btn-save:hover { background: #2ecc71; }
.btn-save:disabled { background: #95a5a6; cursor: not-allowed; }
</style>