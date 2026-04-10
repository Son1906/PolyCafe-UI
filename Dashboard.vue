<template>
  <div class="mgmt-page">
    <div class="header">
      <h2>📊 Tổng Quan Kinh Doanh</h2>
      <button class="btn-refresh" @click="loadData">🔄 Làm mới dữ liệu</button>
    </div>

    <!-- KHU VỰC BỘ LỌC THỜI GIAN -->
    <div class="filter-section">
      <div class="filter-group">
        <label>Từ ngày:</label>
        <input type="date" v-model="filter.startDate" class="date-input" />
      </div>
      <div class="filter-group">
        <label>Đến ngày:</label>
        <input type="date" v-model="filter.endDate" class="date-input" />
      </div>
      <button class="btn-filter" @click="applyFilter">🔍 Lọc Thống Kê</button>
      <button class="btn-clear" @click="clearFilter">Xóa lọc</button>
    </div>

    <!-- TỔNG KẾT DOANH THU THẺ -->
    <div class="summary-cards">
      <div class="card bg-blue">
        <div class="card-icon">💰</div>
        <div class="card-info">
          <p>Tổng Doanh Thu</p>
          <h3>{{ formatPrice(totalRevenue) }}</h3>
        </div>
      </div>
      <div class="card bg-green">
        <div class="card-icon">🧾</div>
        <div class="card-info">
          <p>Đơn Thành Công</p>
          <h3>{{ successOrders }} Đơn</h3>
        </div>
      </div>
      <div class="card bg-red">
        <div class="card-icon">❌</div>
        <div class="card-info">
          <p>Đơn Đã Hủy</p>
          <h3>{{ canceledOrders }} Đơn</h3>
        </div>
      </div>
    </div>

    <div class="content-grid">
      <!-- BIỂU ĐỒ -->
      <div class="chart-container">
        <h3 class="chart-title">📈 Biểu đồ doanh thu (7 ngày gần nhất)</h3>
        <div class="chart-wrapper">
          <Bar v-if="loaded" :data="chartData" :options="chartOptions" />
          <div v-else class="loading-text">Đang tải dữ liệu biểu đồ...</div>
        </div>
      </div>

      <!-- TOP 5 THỨC UỐNG -->
      <div class="top-drinks-container">
        <h3 class="chart-title">🏆 Top 5 Thức Uống Bán Chạy</h3>
        <div class="top-list" v-if="topDrinks.length > 0">
          <div class="top-item" v-for="(drink, index) in topDrinks" :key="index">
            <div class="rank" :class="'rank-' + (index + 1)">#{{ index + 1 }}</div>
            <div class="drink-info">
              <h4>{{ drink.name }}</h4>
              <p>Đã bán: <b>{{ drink.quantity }}</b> ly</p>
            </div>
          </div>
        </div>
        <div class="loading-text" v-else>Không có dữ liệu thức uống trong khoảng thời gian này.</div>
      </div>
    </div>

  </div>
</template>

<script setup>
import { ref, reactive, onMounted } from 'vue';
import api from '../../services/api';

import { Bar } from 'vue-chartjs';
import { Chart as ChartJS, Title, Tooltip, Legend, BarElement, CategoryScale, LinearScale } from 'chart.js';
ChartJS.register(Title, Tooltip, Legend, BarElement, CategoryScale, LinearScale);

const loaded = ref(false);

const totalRevenue = ref(0);
const successOrders = ref(0);
const canceledOrders = ref(0);
const topDrinks = ref([]);
const rawBills = ref([]); 

// Bộ lọc
const filter = reactive({
  startDate: '',
  endDate: ''
});

const chartData = ref({
  labels: [], 
  datasets: [
    { label: 'Doanh thu (VNĐ)', backgroundColor: '#f39c12', borderRadius: 4, data: [] }
  ]
});

const chartOptions = {
  responsive: true, maintainAspectRatio: false,
  plugins: {
    legend: { display: false },
    tooltip: { callbacks: { label: function(context) { return formatPrice(context.raw); } } }
  },
  scales: { y: { beginAtZero: true } }
};

onMounted(() => { loadData(); });

const formatPrice = (p) => new Intl.NumberFormat('vi-VN', { style: 'currency', currency: 'VND' }).format(p || 0);

// Fetch dữ liệu từ API
const loadData = async () => {
  loaded.value = false;
  try {
    const res = await api.get('/bills');
    rawBills.value = res.data.data || [];
    
    await fetchTopDrinks(); // Gọi hàm lấy Top 5 ở đây

    applyFilter(); // Tự động áp dụng tính toán lần đầu
  } catch (err) {
    console.error("Lỗi lấy dữ liệu hóa đơn:", err);
  }
};

// ================= HÀM LẤY TOP 5 TỪ BACKEND =================
const fetchTopDrinks = async () => {
  try {
    const res = await api.get('/bill-details/top-drinks'); 
    const rawTopData = res.data.data || [];
    
    // Map mảng Object[] của Java sang dạng Object để hiển thị
    topDrinks.value = rawTopData.map(item => ({
      name: item[0],     
      quantity: item[1]  
    }));
  } catch (err) {
    console.error("Lỗi tải Top 5 thức uống:", err);
  }
};

// ================= XỬ LÝ LỌC THEO NGÀY =================
const applyFilter = () => {
  let filteredBills = [...rawBills.value];

  // Lọc theo khoảng thời gian nếu có chọn
  if (filter.startDate) {
    const start = new Date(filter.startDate).setHours(0,0,0,0);
    filteredBills = filteredBills.filter(b => new Date(b.createdAt).getTime() >= start);
  }
  if (filter.endDate) {
    const end = new Date(filter.endDate).setHours(23,59,59,999);
    filteredBills = filteredBills.filter(b => new Date(b.createdAt).getTime() <= end);
  }

  // Cập nhật các thẻ thông tin
  const successBills = filteredBills.filter(b => b.billStatus === 'Hoàn thành');
  successOrders.value = successBills.length;
  canceledOrders.value = filteredBills.filter(b => b.billStatus === 'Đã hủy').length;
  totalRevenue.value = successBills.reduce((sum, b) => sum + (b.totalPrice || 0), 0);

  // Xử lý Chart
  processChartData(successBills);
};

const clearFilter = () => {
  filter.startDate = '';
  filter.endDate = '';
  applyFilter();
};

// ================= XỬ LÝ BIỂU ĐỒ (7 Ngày gần nhất) =================
const processChartData = (successBills) => {
  const last7Days = [];
  const revenueData = [];

  for (let i = 6; i >= 0; i--) {
    const d = new Date();
    d.setDate(d.getDate() - i);
    const dateString = d.toISOString().split('T')[0];
    const displayDate = `${d.getDate()}/${d.getMonth() + 1}`;
    last7Days.push({ fullDate: dateString, display: displayDate });
  }

  last7Days.forEach(dayItem => {
    const billsOfDay = successBills.filter(b => {
      if (!b.createdAt) return false;
      return b.createdAt.startsWith(dayItem.fullDate);
    });
    const dayTotal = billsOfDay.reduce((sum, b) => sum + (b.totalPrice || 0), 0);
    revenueData.push(dayTotal);
  });

  chartData.value.labels = last7Days.map(d => d.display);
  chartData.value.datasets[0].data = revenueData;
  loaded.value = true;
};
</script>

<style scoped>
.mgmt-page { background: #f4f6f8; padding: 25px; border-radius: 10px; min-height: 85vh;}
.header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px; }
.header h2 { margin: 0; color: #2c3e50; font-size: 24px; }
.btn-refresh { background: #2c3e50; color: white; border: none; padding: 10px 15px; border-radius: 6px; cursor: pointer; font-weight: bold;}

/* Bộ lọc */
.filter-section { display: flex; align-items: center; gap: 15px; background: white; padding: 15px 20px; border-radius: 10px; margin-bottom: 25px; box-shadow: 0 4px 10px rgba(0,0,0,0.02);}
.filter-group { display: flex; align-items: center; gap: 10px; }
.filter-group label { font-weight: bold; color: #34495e; font-size: 14px; }
.date-input { padding: 8px 12px; border: 1px solid #ddd; border-radius: 5px; outline: none; }
.btn-filter { background: #3498db; color: white; border: none; padding: 8px 20px; border-radius: 5px; cursor: pointer; font-weight: bold;}
.btn-clear { background: #e74c3c; color: white; border: none; padding: 8px 15px; border-radius: 5px; cursor: pointer;}

/* Thẻ tổng kết */
.summary-cards { display: grid; grid-template-columns: repeat(3, 1fr); gap: 20px; margin-bottom: 30px; }
.card { display: flex; align-items: center; padding: 25px; border-radius: 12px; color: white; box-shadow: 0 4px 10px rgba(0,0,0,0.1); }
.bg-blue { background: linear-gradient(135deg, #3498db, #2980b9); }
.bg-green { background: linear-gradient(135deg, #2ecc71, #27ae60); }
.bg-red { background: linear-gradient(135deg, #e74c3c, #c0392b); }
.card-icon { font-size: 40px; margin-right: 20px; background: rgba(255,255,255,0.2); width: 70px; height: 70px; display: flex; align-items: center; justify-content: center; border-radius: 50%; }
.card-info p { margin: 0 0 5px 0; font-size: 14px; opacity: 0.9; font-weight: bold; text-transform: uppercase;}
.card-info h3 { margin: 0; font-size: 28px; }

/* Bố cục lưới cho Biểu đồ và Top 5 */
.content-grid { display: grid; grid-template-columns: 2fr 1fr; gap: 20px; }

/* Chart và Top Drinks container */
.chart-container, .top-drinks-container { background: white; padding: 25px; border-radius: 12px; box-shadow: 0 4px 15px rgba(0,0,0,0.05); }
.chart-title { margin: 0 0 20px 0; color: #34495e; font-size: 18px; border-bottom: 2px solid #f4f6f8; padding-bottom: 10px;}
.chart-wrapper { height: 400px; position: relative; }
.loading-text { text-align: center; color: #7f8c8d; padding-top: 100px; font-style: italic; font-size: 16px;}

/* Danh sách top 5 */
.top-list { display: flex; flex-direction: column; gap: 15px; margin-top: 10px; }
.top-item { display: flex; align-items: center; padding: 12px; background: #f9fbfd; border-radius: 8px; border: 1px solid #ecf0f1; transition: 0.2s;}
.top-item:hover { transform: translateY(-3px); box-shadow: 0 4px 10px rgba(0,0,0,0.05); }
.rank { width: 35px; height: 35px; border-radius: 50%; display: flex; align-items: center; justify-content: center; font-weight: bold; color: white; margin-right: 15px; flex-shrink: 0;}
.rank-1 { background: #f1c40f; } /* Vàng */
.rank-2 { background: #bdc3c7; } /* Bạc */
.rank-3 { background: #cd7f32; } /* Đồng */
.rank-4, .rank-5 { background: #34495e; }

.drink-info h4 { margin: 0 0 5px 0; color: #2c3e50; font-size: 15px;}
.drink-info p { margin: 0; font-size: 13px; color: #7f8c8d; }
.drink-info b { color: #27ae60; }
</style>