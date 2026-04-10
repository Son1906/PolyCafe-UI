<template>
  <div class="mgmt-page">
    <div class="header">
      <h2>📦 Quản Lý Phiếu Nhập Kho</h2>
      <button class="btn-primary" @click="openAddModal">+ Tạo Phiếu Nhập</button>
    </div>

    <table class="table">
      <thead>
        <tr>
          <th>Mã Phiếu (ID)</th>
          <th>Ngày Nhập</th>
          <th>Người Tạo Phiếu</th>
          <th>Tổng Tiền</th>
          <th>Thao tác</th>
        </tr>
      </thead>
      <tbody>
        <tr v-if="receipts.length === 0">
          <td colspan="5" class="text-center">Chưa có phiếu nhập kho nào.</td>
        </tr>
        <tr v-for="receipt in receipts" :key="receipt.irId">
          <td class="code">#{{ receipt.irId }}</td>
          <td>{{ formatDate(receipt.createdAt) }}</td>
          <td>{{ receipt.user?.fullName || `Nhân viên #${receipt.user?.userId}` || 'Hệ thống' }}</td>
          <td class="price">{{ formatPrice(receipt.totalCost) }}</td>
          <td>
            <button class="btn-detail" @click="viewDetails(receipt.irId)">👀 Chi Tiết / Thêm NL</button>
            <button class="btn-delete" @click="deleteReceipt(receipt.irId)">Xóa</button>
          </td>
        </tr>
      </tbody>
    </table>

    <div class="modal-overlay" v-if="showModal">
      <div class="modal-content">
        <h3>✨ Tạo Phiếu Nhập Kho Mới</h3>
        
        <form @submit.prevent="saveReceipt" class="form-grid">
          <div class="form-row">
            <div class="form-group flex-1">
              <label>Tổng chi phí (VNĐ):</label>
              <input type="number" min="0" v-model="formData.totalCost" required />
            </div>
            <div class="form-group flex-1">
              <label>Ngày tạo phiếu:</label>
              <input type="date" v-model="formData.createdAt" required />
            </div>
          </div>
          
          <p class="help-text">💡 Người tạo phiếu sẽ tự động được ghi nhận là bạn ({{ currentUser.fullName }}).</p>

          <div class="modal-actions full-width">
            <button type="button" class="btn-cancel" @click="showModal = false">Hủy</button>
            <button type="submit" class="btn-save" :disabled="loading">
              {{ loading ? 'Đang tạo...' : 'Lưu Phiếu Mới' }}
            </button>
          </div>
        </form>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, reactive, onMounted } from 'vue';
import { useRouter } from 'vue-router';
import api from '../../services/api';

const router = useRouter();
const receipts = ref([]);
const showModal = ref(false);
const loading = ref(false);

// Lấy thông tin người đang đăng nhập
const currentUser = JSON.parse(localStorage.getItem('user')) || { userId: 2, fullName: 'Admin' };

const formData = reactive({
  totalCost: 0,
  createdAt: new Date().toISOString().split('T')[0],
  user: { userId: currentUser.userId } // Gán sẵn ID người dùng
});

onMounted(() => fetchReceipts());

const fetchReceipts = async () => {
  try {
    const res = await api.get('/receipts');
    // Xếp phiếu mới nhất lên đầu
    receipts.value = (res.data.data || []).reverse(); 
  } catch (err) {
    console.error("Lỗi lấy dữ liệu phiếu nhập:", err);
  }
};

const openAddModal = () => {
  Object.assign(formData, {
    totalCost: 0,
    createdAt: new Date().toISOString().split('T')[0],
    user: { userId: currentUser.userId }
  });
  showModal.value = true;
};

// Gọi API POST để lưu phiếu nhập mới
const saveReceipt = async () => {
  loading.value = true;
  try {
    await api.post('/receipts', formData);
    showModal.value = false;
    fetchReceipts();
  } catch (err) {
    alert('Có lỗi xảy ra khi tạo phiếu nhập: ' + (err.response?.data?.message || ''));
  } finally {
    loading.value = false;
  }
};

// CHUYỂN TRANG XEM CHI TIẾT
const viewDetails = (id) => {
  router.push(`/admin/receipts/${id}`);
};

// Lệnh Xóa Phiếu
const deleteReceipt = async (id) => {
  if (confirm("Bạn có chắc chắn muốn xóa Phiếu nhập này không?")) {
    try {
      await api.delete(`/receipts/${id}`);
      fetchReceipts();
    } catch (err) {
      // Bắt lỗi khóa ngoại nếu phiếu này đã có danh sách nguyên liệu bên trong
      alert(err.response?.data?.message || "Không thể xóa phiếu này vì nó đã có dữ liệu chi tiết bên trong. Hãy xóa chi tiết trước!");
    }
  }
};

const formatPrice = (p) => new Intl.NumberFormat('vi-VN', { style: 'currency', currency: 'VND' }).format(p || 0);
const formatDate = (d) => {
  if (!d) return '';
  const date = new Date(d);
  return date.toLocaleDateString('vi-VN');
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

.code { font-weight: bold; color: #8e44ad; }
.price { color: #d35400; font-weight: bold; }
.text-center { text-align: center; color: #95a5a6; font-style: italic; }

.btn-detail { background: #ecf0f1; border: 1px solid #bdc3c7; color: #2c3e50; padding: 8px 12px; border-radius: 6px; cursor: pointer; font-weight: bold; transition: 0.2s; margin-right: 5px;}
.btn-detail:hover { background: #bdc3c7; }
.btn-delete { background: #e74c3c; color: white; border: none; padding: 8px 12px; border-radius: 6px; cursor: pointer; font-weight: bold; transition: 0.2s;}
.btn-delete:hover { background: #c0392b; }

/* MODAL STYLES */
.modal-overlay { position: fixed; top: 0; left: 0; width: 100vw; height: 100vh; background: rgba(0,0,0,0.5); display: flex; justify-content: center; align-items: center; z-index: 1000; }
.modal-content { background: white; padding: 25px; border-radius: 10px; width: 450px; max-width: 90%; box-shadow: 0 10px 30px rgba(0,0,0,0.2);}
.modal-content h3 { margin-top: 0; border-bottom: 1px solid #eee; padding-bottom: 10px; color: #2c3e50;}

.form-grid { display: flex; flex-direction: column; gap: 15px; }
.form-row { display: flex; gap: 15px; }
.flex-1 { flex: 1; }
.form-group label { font-size: 13px; color: #7f8c8d; margin-bottom: 5px; font-weight: bold; display: block;}
.form-group input { width: 100%; padding: 10px; border: 1px solid #bdc3c7; border-radius: 5px; box-sizing: border-box; font-family: inherit;}
.form-group input:focus { outline: none; border-color: #3498db; }

.help-text { font-size: 13px; color: #f39c12; margin: 0; font-style: italic; }

.modal-actions { display: flex; justify-content: flex-end; gap: 10px; margin-top: 20px; }
.btn-cancel { background: #95a5a6; color: white; border: none; padding: 10px 15px; border-radius: 5px; cursor: pointer; font-weight: bold;}
.btn-cancel:hover { background: #7f8c8d; }
.btn-save { background: #27ae60; color: white; border: none; padding: 10px 20px; border-radius: 5px; font-weight: bold; cursor: pointer; }
.btn-save:hover { background: #2ecc71; }
</style>