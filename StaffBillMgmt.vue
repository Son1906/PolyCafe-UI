<template>
  <div class="mgmt-page">
    <div class="header">
      <h2>📦 Quản Lý Đơn Hàng (Dành cho Nhân viên)</h2>
      <div class="filter-bar">
        <span class="summary">Tổng số đơn trong ngày: <strong>{{ bills.length }}</strong></span>
        <button class="btn-refresh" @click="fetchBills">🔄 Cập nhật đơn</button>
      </div>
    </div>

    <table class="table">
      <thead>
        <tr>
          <th>Mã Đơn</th>
          <th>Thời gian</th>
          <th>Loại đơn & Địa chỉ</th>
          <th>Thanh toán</th>
          <th>Tổng tiền</th>
          <th>Xử lý Trạng thái</th>
          <th>Thao tác</th>
        </tr>
      </thead>
      <tbody>
        <tr v-if="bills.length === 0">
          <td colspan="7" class="text-center">Chưa có dữ liệu đơn hàng nào.</td>
        </tr>
        <tr v-for="b in bills" :key="b.billId" :class="{'row-completed': b.billStatus === 'Hoàn thành' || b.billStatus === 'Đã hủy'}">
          <td class="code">{{ b.billCode }}</td>
          <td>{{ formatDate(b.createdAt) }}</td>
          <td>
            <span class="type-badge" :class="{'delivery-badge': b.orderType === 'Giao hàng'}">
              {{ b.orderType || 'Tại quán' }}
            </span>
            <div v-if="b.orderType === 'Giao hàng'" class="address-text">
              📍 {{ b.deliveryAddress }}
            </div>
          </td>
          <td>{{ b.paymentMethod }}</td>
          <td class="price">{{ formatPrice(b.totalPrice) }}</td>
          <td>
            <select 
              v-model="b.billStatus" 
              @change="handleStatusChange(b, $event)" 
              class="status-select" 
              :class="getStatusClass(b.billStatus)"
              :disabled="b.billStatus === 'Hoàn thành' || b.billStatus === 'Đã hủy'"
            >
              <option value="Chờ xác nhận">Chờ xác nhận (Web)</option>
              <option value="Chờ pha chế">Chờ pha chế (Bếp)</option>
              <option value="Chờ khách lấy">Chờ khách lấy (Tại quán)</option>
              <!-- TRẠNG THÁI MỚI: SHIPPER ĐÃ TỚI -->
              <option value="Shipper đã tới">Shipper đã tới (Đợi nhận món)</option>
              <option value="Đang giao hàng">Đang giao hàng (Shipper)</option>
              <option value="Hoàn thành">✅ Hoàn thành</option>
              <option value="Đã hủy">❌ Đã hủy</option>
            </select>
            <div class="cancel-reason" v-if="b.billStatus === 'Đã hủy' && b.cancellationReason">
              Lý do: {{ b.cancellationReason }}
            </div>
          </td>
          <td>
            <button class="btn-detail" @click="viewDetails(b)">👀 Chi tiết</button>
          </td>
        </tr>
      </tbody>
    </table>

    <!-- MODAL CHI TIẾT -->
    <div class="modal-overlay" v-if="showModal">
      <div class="modal-content modal-lg">
        <div class="modal-header">
          <h3>Chi Tiết Đơn Hàng <span class="highlight">{{ selectedBill?.billCode }}</span></h3>
          <button class="btn-close" @click="showModal = false">❌</button>
        </div>
        
        <div class="bill-info">
          <p><strong>Ngày giờ:</strong> {{ formatDate(selectedBill?.createdAt) }}</p>
          <p><strong>Loại đơn:</strong> {{ selectedBill?.orderType }} | <strong>Thanh toán:</strong> {{ selectedBill?.paymentMethod }}</p>
          <p><strong>Mã giảm giá áp dụng:</strong> {{ selectedBill?.voucher?.voucherCode || 'Không có' }}</p>
          <p v-if="selectedBill?.orderType === 'Giao hàng'"><strong>Địa chỉ giao:</strong> {{ selectedBill?.deliveryAddress }}</p>
        </div>

        <table class="table details-table">
          <thead>
            <tr>
              <th>Tên món</th>
              <th>Đơn giá</th>
              <th>SL</th>
              <th>Thành tiền</th>
            </tr>
          </thead>
          <tbody>
            <tr v-if="loadingDetails">
              <td colspan="4" class="text-center">Đang tải chi tiết món...</td>
            </tr>
            <tr v-else-if="billDetails.length === 0">
              <td colspan="4" class="text-center">Không tìm thấy chi tiết món.</td>
            </tr>
            <tr v-for="detail in billDetails" :key="detail.bdId">
              <td>
                <b>{{ detail.drink?.drinkName || 'Đồ uống' }}</b>
                <div class="toppings" v-if="detail.toppings && detail.toppings.length">
                  + {{ detail.toppings.map(t => t.topping?.toppingName || t.toppingName).join(', ') }}
                </div>
                <div class="note" v-if="detail.note">📝 {{ detail.note }}</div>
              </td>
              <td>{{ formatPrice(detail.unitPrice) }}</td>
              <td>x{{ detail.bdQuantity || detail.quantity }}</td>
              <td class="price">{{ formatPrice(detail.unitPrice * (detail.bdQuantity || detail.quantity)) }}</td>
            </tr>
          </tbody>
        </table>

        <div class="modal-footer">
          <div class="total-summary">
            <h3>Khách phải trả: <span class="price-large">{{ formatPrice(selectedBill?.totalPrice) }}</span></h3>
          </div>
        </div>
      </div>
    </div>

  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue';
import api from '../../services/api';

const bills = ref([]);
const showModal = ref(false);
const selectedBill = ref(null);
const billDetails = ref([]);
const loadingDetails = ref(false);

onMounted(() => {
  fetchBills();
});

const fetchBills = async () => {
  try {
    const res = await api.get('/bills');
    let allBills = res.data.data || [];
    
    allBills.sort((a, b) => {
      const isACompleted = a.billStatus === 'Hoàn thành' || a.billStatus === 'Đã hủy';
      const isBCompleted = b.billStatus === 'Hoàn thành' || b.billStatus === 'Đã hủy';
      
      if (isACompleted && !isBCompleted) return 1; 
      if (!isACompleted && isBCompleted) return -1; 
      
      return new Date(b.createdAt) - new Date(a.createdAt);
    });

    bills.value = allBills;
  } catch (err) {
    console.error("Lỗi lấy Hóa đơn:", err);
  }
};

const handleStatusChange = async (bill, event) => {
  const newStatus = event.target.value;
  let cancelReason = '';

  if (newStatus === 'Đã hủy') {
    cancelReason = prompt("Vui lòng nhập lý do hủy đơn hàng này (VD: Hết món, Khách đổi ý...):");
    if (cancelReason === null || cancelReason.trim() === '') {
      event.target.value = bill.billStatus;
      fetchBills(); 
      return;
    }
  }

  if (confirm(`Xác nhận đổi trạng thái đơn ${bill.billCode} thành: ${newStatus}?`)) {
    try {
      await api.put(`/bills/${bill.billId}/status`, null, {
        params: { status: newStatus, cancelReason: cancelReason }
      });
      await fetchBills(); 
    } catch (err) {
      alert("Cập nhật trạng thái thất bại! " + (err.response?.data?.message || ""));
      fetchBills(); 
    }
  } else {
    fetchBills(); 
  }
};

const viewDetails = async (bill) => {
  selectedBill.value = bill;
  showModal.value = true;
  billDetails.value = [];
  loadingDetails.value = true;

  try {
    const res = await api.get(`/bill-details/bill/${bill.billId}`);
    const detailsData = res.data.data || [];
    
    for (let detail of detailsData) {
      try {
         const topRes = await api.get(`/bill-detail-toppings/detail/${detail.bdId}`);
         detail.toppings = topRes.data.data || [];
      } catch (e) {
         detail.toppings = [];
      }
    }
    billDetails.value = detailsData;
  } catch (err) {
    console.error("Lỗi lấy chi tiết món:", err);
  } finally {
    loadingDetails.value = false;
  }
};

const formatPrice = (price) => new Intl.NumberFormat('vi-VN', { style: 'currency', currency: 'VND' }).format(price || 0);

const formatDate = (dateStr) => {
  if (!dateStr) return 'N/A';
  return new Date(dateStr).toLocaleString('vi-VN', { day: '2-digit', month: '2-digit', year: 'numeric', hour: '2-digit', minute: '2-digit' });
};

// CẬP NHẬT THÊM CLASS MÀU SẮC CHO "Shipper đã tới"
const getStatusClass = (status) => {
  if (status === 'Đã hủy') return 'danger';
  if (status === 'Chờ xác nhận') return 'secondary';
  if (status === 'Chờ pha chế') return 'warning';
  if (status === 'Đang giao hàng' || status === 'Chờ khách lấy' || status === 'Shipper đã tới') return 'info';
  if (status === 'Hoàn thành') return 'success';
  return 'secondary';
};
</script>

<style scoped>
.mgmt-page { background: white; padding: 25px; border-radius: 10px; box-shadow: 0 4px 15px rgba(0,0,0,0.05); }
.header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px; border-bottom: 2px solid #f4f6f8; padding-bottom: 15px;}
.header h2 { margin: 0; color: #2c3e50; }
.filter-bar { display: flex; align-items: center; gap: 15px; }
.summary { font-size: 15px; color: #7f8c8d; background: #ecf0f1; padding: 8px 15px; border-radius: 20px;}
.btn-refresh { background: #3498db; color: white; border: none; padding: 8px 15px; border-radius: 6px; cursor: pointer; font-weight: bold; transition: 0.2s;}
.btn-refresh:hover { background: #2980b9; }

.table { width: 100%; border-collapse: collapse; text-align: left; }
.table th, .table td { padding: 15px; border-bottom: 1px solid #eee; vertical-align: middle; }
.table th { background: #f8f9fa; color: #34495e; font-weight: 600; text-transform: uppercase; font-size: 13px; letter-spacing: 0.5px;}
.table tr:hover { background: #fdfdfd; }

.row-completed { opacity: 0.6; background: #fafafa; }
.row-completed:hover { opacity: 1; }

.code { font-weight: bold; color: #8e44ad; }
.price { color: #d35400; font-weight: bold; font-size: 16px;}
.text-center { text-align: center; color: #95a5a6; font-style: italic; }

.type-badge { background: #e8f4f8; color: #2980b9; padding: 4px 10px; border-radius: 4px; font-size: 13px; font-weight: bold;}
.delivery-badge { background: #fdf2e9; color: #e67e22; }
.address-text { font-size: 12px; color: #7f8c8d; margin-top: 5px; font-style: italic; max-width: 200px; line-height: 1.3;}

.status-select { padding: 6px 10px; border-radius: 20px; font-size: 13px; font-weight: bold; cursor: pointer; border: 1px solid transparent; outline: none; transition: 0.2s; -webkit-appearance: none; -moz-appearance: none; appearance: none; text-align: center; }
.status-select:disabled { cursor: not-allowed; opacity: 0.8;}
.status-select.success { background: #eafaf1; color: #27ae60; border-color: #27ae60;}
.status-select.warning { background: #fef9e7; color: #f39c12; border-color: #f39c12;}
.status-select.danger { background: #fdedec; color: #e74c3c; border-color: #e74c3c;}
.status-select.info { background: #ebf5fb; color: #2980b9; border-color: #2980b9;}
.status-select.secondary { background: #f4f6f8; color: #7f8c8d; border-color: #bdc3c7;}

.cancel-reason { font-size: 11px; color: #e74c3c; margin-top: 5px; font-style: italic; max-width: 150px;}

.btn-detail { background: #ecf0f1; border: 1px solid #bdc3c7; color: #2c3e50; padding: 8px 12px; border-radius: 6px; cursor: pointer; font-weight: bold; transition: 0.2s;}
.btn-detail:hover { background: #bdc3c7; }

/* MODAL STYLES */
.modal-overlay { position: fixed; top: 0; left: 0; width: 100vw; height: 100vh; background: rgba(0,0,0,0.5); display: flex; justify-content: center; align-items: center; z-index: 1000; }
.modal-content { background: white; padding: 0; border-radius: 12px; width: 600px; max-width: 95%; box-shadow: 0 10px 30px rgba(0,0,0,0.2); overflow: hidden; display: flex; flex-direction: column; max-height: 90vh;}
.modal-header { display: flex; justify-content: space-between; align-items: center; padding: 20px 25px; background: #f8f9fa; border-bottom: 1px solid #eee; }
.modal-header h3 { margin: 0; color: #2c3e50; }
.highlight { color: #e67e22; }
.btn-close { background: none; border: none; font-size: 20px; cursor: pointer; color: #7f8c8d; transition: 0.2s;}
.btn-close:hover { transform: scale(1.1); color: #e74c3c; }

.bill-info { padding: 20px 25px; background: #fff; border-bottom: 1px dashed #ccc; display: grid; grid-template-columns: 1fr 1fr; gap: 10px; font-size: 14px; }
.bill-info p { margin: 0; color: #34495e; }

.details-table { margin: 0; }
.details-table th, .details-table td { padding: 12px 25px; }
.toppings { font-size: 12px; color: #7f8c8d; margin-top: 4px; }
.note { font-size: 12px; color: #e67e22; font-style: italic; margin-top: 4px; }

.modal-footer { padding: 20px 25px; background: #f8f9fa; border-top: 1px solid #eee; display: flex; justify-content: flex-end; }
.total-summary h3 { margin: 0; color: #2c3e50; }
.price-large { color: #c0392b; font-size: 24px; }
</style>