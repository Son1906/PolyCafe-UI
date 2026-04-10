<template>
  <div class="mgmt-page">
    <div class="header">
      <div class="title-area">
        <h2>📊 Thống Kê & Doanh Thu</h2>
        <p class="subtitle">Theo dõi hiệu quả kinh doanh của quán</p>
      </div>

      <!-- BỘ LỌC THỜI GIAN -->
      <div class="filter-group">
        <label>Hiển thị theo:</label>
        <select v-model="filterType" class="filter-select">
          <option value="today">Hôm nay</option>
          <option value="week">Tuần này (7 ngày qua)</option>
          <option value="month">Tháng này</option>
          <option value="year">Năm nay</option>
          <option value="all">Tất cả thời gian</option>
        </select>
        <button class="btn-refresh" @click="fetchBills" :disabled="loading">
          {{ loading ? '⏳' : '🔄' }}
        </button>
      </div>
    </div>

    <!-- CÁC THẺ TỔNG QUAN (SUMMARY CARDS) -->
    <div class="summary-cards">
      <div class="card card-revenue">
        <div class="card-icon">💰</div>
        <div class="card-info">
          <p>Tổng Doanh Thu</p>
          <h3>{{ formatPrice(totalRevenue) }}</h3>
        </div>
      </div>

      <div class="card card-success">
        <div class="card-icon">✅</div>
        <div class="card-info">
          <p>Đơn Thành Công</p>
          <h3>{{ successOrdersCount }} <small>đơn</small></h3>
        </div>
      </div>

      <div class="card card-danger">
        <div class="card-icon">❌</div>
        <div class="card-info">
          <p>Đơn Đã Hủy</p>
          <h3>{{ canceledOrdersCount }} <small>đơn</small></h3>
        </div>
      </div>
    </div>

    <!-- DANH SÁCH CHI TIẾT CÁC ĐƠN TRONG KHOẢNG THỜI GIAN ĐÃ LỌC -->
    <div class="details-section">
      <h3>Chi Tiết Đơn Hàng ({{ filterText }})</h3>
      
      <table class="table">
        <thead>
          <tr>
            <th>Mã Đơn</th>
            <th>Ngày tạo</th>
            <th>Loại đơn</th>
            <th>Trạng thái</th>
            <th>Tổng tiền</th>
          </tr>
        </thead>
        <tbody>
          <tr v-if="filteredBills.length === 0">
            <td colspan="5" class="text-center">Không có đơn hàng nào trong khoảng thời gian này.</td>
          </tr>
          <tr v-for="b in filteredBills" :key="b.billId">
            <td class="code">{{ b.billCode }}</td>
            <td>{{ formatDate(b.createdAt) }}</td>
            <td>
              <span class="type-badge">{{ b.orderType || 'Tại quán' }}</span>
            </td>
            <td>
              <span class="status-badge" :class="getStatusClass(b.billStatus)">
                {{ b.billStatus }}
              </span>
            </td>
            <td class="price">{{ formatPrice(b.totalPrice) }}</td>
          </tr>
        </tbody>
      </table>
    </div>

  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue';
import api from '../../services/api';

const bills = ref([]);
const loading = ref(false);

// Biến lưu trữ loại lọc thời gian (Mặc định là 'month')
const filterType = ref('month');

onMounted(() => {
  fetchBills();
});

const fetchBills = async () => {
  loading.value = true;
  try {
    const res = await api.get('/bills');
    // Sắp xếp đơn mới nhất lên đầu
    bills.value = (res.data.data || []).sort((a, b) => new Date(b.createdAt) - new Date(a.createdAt));
  } catch (err) {
    console.error("Lỗi lấy dữ liệu hóa đơn:", err);
  } finally {
    loading.value = false;
  }
};

// ================= LOGIC LỌC THỜI GIAN =================
const filteredBills = computed(() => {
  const now = new Date();
  
  return bills.value.filter(bill => {
    if (!bill.createdAt) return false;
    const billDate = new Date(bill.createdAt);

    switch (filterType.value) {
      case 'today':
        return billDate.toDateString() === now.toDateString();
        
      case 'week':
        // Lấy các đơn trong vòng 7 ngày qua
        const sevenDaysAgo = new Date();
        sevenDaysAgo.setDate(now.getDate() - 7);
        return billDate >= sevenDaysAgo && billDate <= now;
        
      case 'month':
        return billDate.getMonth() === now.getMonth() && billDate.getFullYear() === now.getFullYear();
        
      case 'year':
        return billDate.getFullYear() === now.getFullYear();
        
      case 'all':
      default:
        return true;
    }
  });
});

// ================= TÍNH TOÁN CÁC CHỈ SỐ =================

// Tính TỔNG DOANH THU (Chỉ tính những đơn "Hoàn thành")
const totalRevenue = computed(() => {
  return filteredBills.value
    .filter(b => b.billStatus === 'Hoàn thành')
    .reduce((sum, b) => sum + (b.totalPrice || 0), 0);
});

// Tính SỐ ĐƠN THÀNH CÔNG
const successOrdersCount = computed(() => {
  return filteredBills.value.filter(b => b.billStatus === 'Hoàn thành').length;
});

// Tính SỐ ĐƠN ĐÃ HỦY
const canceledOrdersCount = computed(() => {
  return filteredBills.value.filter(b => b.billStatus === 'Đã hủy').length;
});

// Text hiển thị cho tiêu đề bảng
const filterText = computed(() => {
  const map = {
    today: 'Hôm nay',
    week: '7 ngày qua',
    month: 'Tháng này',
    year: 'Năm nay',
    all: 'Tất cả thời gian'
  };
  return map[filterType.value];
});

// ================= TIỆN ÍCH HIỂN THỊ =================
const formatPrice = (price) => new Intl.NumberFormat('vi-VN', { style: 'currency', currency: 'VND' }).format(price || 0);

const formatDate = (dateStr) => {
  if (!dateStr) return 'N/A';
  return new Date(dateStr).toLocaleString('vi-VN', { day: '2-digit', month: '2-digit', year: 'numeric', hour: '2-digit', minute: '2-digit' });
};

const getStatusClass = (status) => {
  if (status === 'Đã hủy') return 'danger';
  if (status === 'Hoàn thành') return 'success';
  return 'warning'; // Các trạng thái chờ khác
};
</script>

<style scoped>
.mgmt-page { background: #f8f9fa; padding: 25px; border-radius: 10px; min-height: 100vh;}
.header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 30px; background: white; padding: 20px; border-radius: 10px; box-shadow: 0 2px 10px rgba(0,0,0,0.05);}
.title-area h2 { margin: 0; color: #2c3e50; font-size: 24px;}
.subtitle { margin: 5px 0 0 0; color: #7f8c8d; font-size: 14px;}

.filter-group { display: flex; align-items: center; gap: 15px; background: #f4f6f8; padding: 10px 20px; border-radius: 8px; border: 1px solid #e0e6ed;}
.filter-group label { font-weight: bold; color: #34495e; font-size: 14px;}
.filter-select { padding: 8px 15px; border: 1px solid #bdc3c7; border-radius: 5px; font-weight: bold; color: #2c3e50; outline: none; cursor: pointer; background: white;}
.filter-select:focus { border-color: #3498db; }
.btn-refresh { background: white; border: 1px solid #bdc3c7; border-radius: 5px; width: 35px; height: 35px; display: flex; align-items: center; justify-content: center; cursor: pointer; transition: 0.2s;}
.btn-refresh:hover { background: #ecf0f1; transform: rotate(15deg);}

/* SUMMARY CARDS */
.summary-cards { display: grid; grid-template-columns: repeat(3, 1fr); gap: 20px; margin-bottom: 30px;}
.card { display: flex; align-items: center; padding: 25px; border-radius: 12px; box-shadow: 0 4px 15px rgba(0,0,0,0.05); color: white; transition: 0.3s;}
.card:hover { transform: translateY(-5px); box-shadow: 0 8px 20px rgba(0,0,0,0.1);}
.card-icon { font-size: 45px; margin-right: 20px; background: rgba(255,255,255,0.2); width: 80px; height: 80px; display: flex; align-items: center; justify-content: center; border-radius: 50%;}
.card-info p { margin: 0 0 5px 0; font-size: 16px; opacity: 0.9; font-weight: 600; text-transform: uppercase;}
.card-info h3 { margin: 0; font-size: 32px; letter-spacing: 1px;}
.card-info small { font-size: 16px; font-weight: normal; opacity: 0.8;}

.card-revenue { background: linear-gradient(135deg, #27ae60, #2ecc71); }
.card-success { background: linear-gradient(135deg, #2980b9, #3498db); }
.card-danger { background: linear-gradient(135deg, #c0392b, #e74c3c); }

/* DETAILS SECTION */
.details-section { background: white; padding: 25px; border-radius: 12px; box-shadow: 0 4px 15px rgba(0,0,0,0.05);}
.details-section h3 { margin: 0 0 20px 0; color: #2c3e50; border-bottom: 2px solid #f4f6f8; padding-bottom: 10px;}

.table { width: 100%; border-collapse: collapse; text-align: left; }
.table th, .table td { padding: 15px; border-bottom: 1px solid #eee; vertical-align: middle; }
.table th { background: #f8f9fa; color: #34495e; font-weight: 600; text-transform: uppercase; font-size: 13px; letter-spacing: 0.5px;}
.table tr:hover { background: #fdfdfd; }

.code { font-weight: bold; color: #8e44ad; }
.price { color: #d35400; font-weight: bold; font-size: 15px;}
.text-center { text-align: center; color: #95a5a6; font-style: italic; }

.type-badge { background: #e8f4f8; color: #2980b9; padding: 4px 10px; border-radius: 4px; font-size: 12px; font-weight: bold;}

.status-badge { padding: 5px 12px; border-radius: 20px; font-size: 12px; font-weight: bold;}
.status-badge.success { background: #eafaf1; color: #27ae60; border: 1px solid #27ae60;}
.status-badge.danger { background: #fdedec; color: #e74c3c; border: 1px solid #e74c3c;}
.status-badge.warning { background: #fef9e7; color: #f39c12; border: 1px solid #f39c12;}
</style>