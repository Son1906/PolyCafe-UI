<template>
  <div class="mgmt-page">
    <div class="header">
      <div class="title-area">
        <h2>💰 Quản Lý Tiền Lương & Hiệu Suất</h2>
        <p class="subtitle">Tính toán thu nhập dựa trên hoạt động của nhân viên</p>
      </div>
      <div class="filter-group">
        <label>Tháng lương:</label>
        <select v-model="selectedMonth" class="filter-select">
          <option v-for="m in 12" :key="m" :value="m">Tháng {{ m }}</option>
        </select>
        <button class="btn-refresh" @click="calculatePayroll">🔄 Tính toán lại</button>
      </div>
    </div>

    <!-- CẤU HÌNH MỨC LƯƠNG -->
    <div class="config-section">
      <h3>⚙️ Cấu hình mức lương (VNĐ/Giờ)</h3>
      <div class="config-grid">
        <div class="config-item">
          <label>Nhân viên phục vụ (Staff):</label>
          <input type="number" v-model="salaryConfig.staffRate" @change="saveConfig" />
        </div>
        <div class="config-item">
          <label>Quản lý kho (Warehouse):</label>
          <input type="number" v-model="salaryConfig.warehouseRate" @change="saveConfig" />
        </div>
        <p class="note">💡 Hệ thống tính: 1 đơn hoàn thành = 20 phút làm việc.</p>
      </div>
    </div>

    <!-- BẢNG LƯƠNG CHI TIẾT -->
    <div class="salary-table-container">
      <table class="table">
        <thead>
          <tr>
            <th>Nhân viên</th>
            <th>Vai trò</th>
            <th>Số đơn đã xử lý</th>
            <th>Ước tính giờ làm</th>
            <th>Lương tạm tính</th>
            <th>Thưởng/Phụ cấp</th>
            <th>Tổng lĩnh</th>
            <th>Thao tác</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="staff in payrollData" :key="staff.userId">
            <td class="staff-name">
              <b>{{ staff.fullName }}</b>
              <small>{{ staff.email }}</small>
            </td>
            <td>
              <span class="role-badge" :class="'role-' + staff.role">
                {{ getRoleName(staff.role) }}
              </span>
            </td>
            <td class="text-center">{{ staff.orderCount }} đơn</td>
            <td class="text-center">{{ (staff.orderCount * 20 / 60).toFixed(1) }} giờ</td>
            <td class="price">{{ formatPrice(calculateBaseSalary(staff)) }}</td>
            <td>
              <input type="number" v-model="staff.bonus" class="inline-input" placeholder="0" />
            </td>
            <td class="total-salary">{{ formatPrice(calculateBaseSalary(staff) + (staff.bonus || 0)) }}</td>
            <td>
              <button class="btn-pay" @click="confirmPayment(staff)">💳 Thanh toán</button>
            </td>
          </tr>
        </tbody>
      </table>
    </div>
  </div>
</template>

<script setup>
import { ref, reactive, onMounted } from 'vue';
import api from '../../services/api';

const selectedMonth = ref(new Date().getMonth() + 1);
const payrollData = ref([]);
const users = ref([]);
const bills = ref([]);

const salaryConfig = reactive({
  staffRate: JSON.parse(localStorage.getItem('staffRate')) || 25000,
  warehouseRate: JSON.parse(localStorage.getItem('warehouseRate')) || 28000
});

onMounted(async () => {
  await loadBaseData();
  calculatePayroll();
});

const loadBaseData = async () => {
  try {
    const [resUsers, resBills] = await Promise.all([
      api.get('/users'),
      api.get('/bills')
    ]);
    // Chỉ lấy Staff (Role 1) và Warehouse (Role 3)
    users.value = (resUsers.data.data || []).filter(u => u.role !== 2);
    bills.value = resBills.data.data || [];
  } catch (e) {
    console.error("Lỗi tải dữ liệu", e);
  }
};

const calculatePayroll = () => {
  payrollData.value = users.value.map(user => {
    // Đếm số đơn nhân viên này đã hoàn thành trong tháng được chọn
    const userOrders = bills.value.filter(bill => {
      const billDate = new Date(bill.createdAt);
      return bill.user?.userId === user.userId && 
             bill.billStatus === 'Hoàn thành' &&
             (billDate.getMonth() + 1) === selectedMonth.value;
    });

    return {
      ...user,
      orderCount: userOrders.length,
      bonus: 0
    };
  });
};

const calculateBaseSalary = (staff) => {
  const rate = staff.role === 3 ? salaryConfig.warehouseRate : salaryConfig.staffRate;
  const hours = (staff.orderCount * 20) / 60; // 20 phút mỗi đơn
  return Math.round(hours * rate);
};

const saveConfig = () => {
  localStorage.setItem('staffRate', salaryConfig.staffRate);
  localStorage.setItem('warehouseRate', salaryConfig.warehouseRate);
};

const confirmPayment = (staff) => {
  const total = calculateBaseSalary(staff) + (staff.bonus || 0);
  alert(`Đã xác nhận thanh toán lương Tháng ${selectedMonth.value} cho ${staff.fullName}\nSố tiền: ${formatPrice(total)}`);
};

const getRoleName = (role) => {
  if (role === 1) return 'Nhân viên (Staff)';
  if (role === 3) return 'Kho (Warehouse)';
  return 'Khác';
};

const formatPrice = (p) => new Intl.NumberFormat('vi-VN', { style: 'currency', currency: 'VND' }).format(p || 0);
</script>

<style scoped>
.mgmt-page { background: #f8f9fa; padding: 25px; border-radius: 12px; }
.header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 25px; }
.title-area h2 { margin: 0; color: #2c3e50; }
.subtitle { color: #7f8c8d; font-size: 14px; margin: 5px 0 0 0; }

.config-section { background: white; padding: 20px; border-radius: 10px; margin-bottom: 25px; box-shadow: 0 2px 10px rgba(0,0,0,0.05); }
.config-grid { display: flex; gap: 30px; align-items: center; margin-top: 15px; }
.config-item { display: flex; align-items: center; gap: 10px; }
.config-item input { width: 100px; padding: 8px; border: 1px solid #ddd; border-radius: 5px; font-weight: bold; color: #27ae60; }
.note { color: #f39c12; font-size: 13px; font-style: italic; }

.salary-table-container { background: white; border-radius: 10px; padding: 20px; box-shadow: 0 2px 10px rgba(0,0,0,0.05); }
.table { width: 100%; border-collapse: collapse; }
.table th { text-align: left; padding: 15px; background: #f8f9fa; color: #34495e; font-size: 13px; text-transform: uppercase; }
.table td { padding: 15px; border-bottom: 1px solid #eee; }

.staff-name b { display: block; color: #2c3e50; }
.staff-name small { color: #7f8c8d; }

.role-badge { padding: 4px 8px; border-radius: 4px; font-size: 11px; font-weight: bold; }
.role-1 { background: #e8f4f8; color: #2980b9; }
.role-3 { background: #fef9e7; color: #f39c12; }

.price { color: #2c3e50; font-weight: 600; }
.total-salary { color: #c0392b; font-weight: bold; font-size: 16px; }

.inline-input { width: 120px; padding: 6px; border: 1px solid #ccc; border-radius: 4px; }
.btn-pay { background: #27ae60; color: white; border: none; padding: 8px 15px; border-radius: 5px; cursor: pointer; font-weight: bold; }
.btn-pay:hover { background: #2ecc71; }

.text-center { text-align: center; }
</style>