<template>
  <div class="kds-page">
    <div class="header">
      <div style="display: flex; align-items: center; gap: 15px;">
        <button v-if="currentUser && currentUser.role == 2" @click="$router.push('/admin')" class="btn-return-admin">
          🔙 Quản Trị
        </button>
        <h2>🔥 Màn Hình Bếp (Chờ Pha Chế)</h2>
      </div>
      
      <div class="header-right">
        <span class="timer-text">Cập nhật tự động mỗi 10s...</span>
        <button class="btn-refresh" @click="fetchPendingOrders" :disabled="loading">
          {{ loading ? '⏳ Đang quét...' : '🔄 Làm mới ngay' }}
        </button>
      </div>
    </div>

    <div class="kds-container">
      
      <div class="kds-column">
        <h3 class="col-title text-orange">ĐANG CHỜ PHA CHẾ ({{ pendingBills.length }})</h3>
        
        <div v-if="pendingBills.length === 0 && !loading" class="empty-state">
          <h2>🎉 Tuyệt vời!</h2>
          <p>Hiện không có đơn nào đang chờ pha chế.</p>
        </div>

        <div class="kds-grid">
          <div class="ticket ticket-pending" v-for="bill in pendingBills" :key="bill.billId">
            <div class="head">
              <span class="code">{{ bill.billCode }}</span>
              <span class="badge" :class="bill.orderType === 'Giao hàng' ? 'delivery' : (bill.orderType === 'Tại quán' ? 'in-store' : 'take-away')">
                {{ bill.orderType || 'Tại quán' }}
              </span>
            </div>

            <!-- HIỂN THỊ THẺ RUNG (SHIPPER HOẶC OFFLINE) -->
            <div class="pager-alert" :class="{'is-shipper': bill.orderType === 'Giao hàng'}">
              <span v-if="bill.orderType === 'Giao hàng'">
                🛵 <b>SHIPPER</b> - Đối chiếu mã: <b>{{ bill.billCode }}</b>
              </span>
              <span v-else>
                🔔 Gọi Khách: <b>{{ bill.pagerCode ? 'Thẻ số ' + bill.pagerCode : 'Mã ' + bill.billCode }}</b>
              </span>
            </div>
            
            <div class="time">⏱️ Đặt lúc: {{ formatTime(bill.createdAt) }}</div>

            <ul class="items-list">
              <li v-for="detail in bill.details" :key="detail.bdId">
                <b class="qty">{{ detail.bdQuantity }}x</b> 
                <span class="drink-name">{{ detail.drink?.drinkName }}</span>
                
                <div class="toppings" v-if="detail.toppings && detail.toppings.length">
                  + {{ detail.toppings.map(t => t.topping?.toppingName).join(', ') }}
                </div>
              </li>
            </ul>

            <button class="btn-done" @click="markAsReady(bill)">✔️ XONG, CHUYỂN CHỜ LẤY</button>
          </div>
        </div>
      </div>

      <div class="kds-column column-ready">
        <h3 class="col-title text-green">CHỜ KHÁCH LẤY NƯỚC ({{ readyBills.length }})</h3>
        
        <div v-if="readyBills.length === 0" class="empty-state">
          <p>Chưa có đơn nào chờ lấy.</p>
        </div>

        <div class="kds-grid ready-grid">
          <div class="ticket ticket-ready" v-for="bill in readyBills" :key="bill.billId">
            <div class="head">
              <span class="code">{{ bill.billCode }}</span>
            </div>

            <div class="pager-alert small-alert" :class="{'is-shipper': bill.orderType === 'Giao hàng'}">
               <span v-if="bill.orderType === 'Giao hàng'">🛵 Gọi Shipper (Mã: {{ bill.billCode }})</span>
               <span v-else>🔔 Gọi: <b>{{ bill.pagerCode ? 'Thẻ ' + bill.pagerCode : 'Mã ' + bill.billCode }}</b></span>
            </div>

            <div class="time" style="text-align: left; margin-top: 5px;">Món: {{ bill.details?.length }} ly</div>
            
            <!-- NÚT BẤM THÔNG MINH DỰA TRÊN LOẠI ĐƠN -->
            <button 
              v-if="bill.orderType === 'Giao hàng'" 
              class="btn-finish delivery-btn" 
              @click="markAsShipped(bill)"
            >
              🛵 ĐƯA SHIPPER ĐI GIAO
            </button>

            <button 
              v-else 
              class="btn-finish" 
              @click="markAsCompleted(bill)"
            >
              ✅ GIAO KHÁCH (HOÀN THÀNH)
            </button>

          </div>
        </div>
      </div>

    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue';
import api from '../../services/api';

const pendingBills = ref([]); 
const readyBills = ref([]);   
const loading = ref(false);
let refreshInterval = null;

const currentUser = JSON.parse(localStorage.getItem('user'));

onMounted(() => {
  fetchPendingOrders();
  refreshInterval = setInterval(() => { fetchPendingOrders(true); }, 10000); 
});

onUnmounted(() => { clearInterval(refreshInterval); });

const fetchPendingOrders = async (isSilent = false) => {
  if (!isSilent) loading.value = true;
  try {
    const res = await api.get('/bills');
    const allBills = res.data.data || [];
    
    // 1. Lọc đơn chờ pha chế
    let pending = allBills.filter(b => b.billStatus === 'Chờ pha chế' || b.billStatus === 'Đang xử lý');
    
    // THUẬT TOÁN: Ưu tiên đơn Shipper (Giao hàng) lên đầu tiên
    pending.sort((a, b) => {
      const isAShipper = (a.orderType === 'Giao hàng');
      const isBShipper = (b.orderType === 'Giao hàng');
      
      if (isAShipper && !isBShipper) return -1; // Đẩy A lên trước
      if (!isAShipper && isBShipper) return 1;  // Đẩy B lên trước
      // Cùng loại thì ưu tiên đơn đặt trước (cũ hơn)
      return new Date(a.createdAt) - new Date(b.createdAt);
    });
    
    // 2. Lọc đơn chờ khách lấy
    const ready = allBills.filter(b => b.billStatus === 'Chờ khách lấy');
    
    for (let bill of pending) {
      const detRes = await api.get(`/bill-details/bill/${bill.billId}`);
      const detailsArray = detRes.data.data || [];
      
      for (let detail of detailsArray) {
        try {
           const topRes = await api.get(`/bill-detail-toppings/detail/${detail.bdId}`);
           detail.toppings = topRes.data.data || [];
        } catch (e) { detail.toppings = []; }
      }
      bill.details = detailsArray;
    }

    for (let bill of ready) {
       const detRes = await api.get(`/bill-details/bill/${bill.billId}`);
       bill.details = detRes.data.data || [];
    }
    
    pendingBills.value = pending;
    readyBills.value = ready;
  } catch (err) { console.error("Lỗi lấy đơn KDS:", err); } 
  finally { loading.value = false; }
};

const markAsReady = async (bill) => {
  if(confirm(`Đã pha chế xong đơn ${bill.billCode}? Chuyển sang đợi khách lấy?`)) {
    try {
      const payload = { ...bill, billStatus: 'Chờ khách lấy' };
      delete payload.details; 
      await api.put(`/bills/${bill.billId}`, payload);
      fetchPendingOrders();
    } catch (err) { alert("Lỗi cập nhật!"); }
  }
};

// Khách Offline -> Bấm hoàn thành
const markAsCompleted = async (bill) => {
  try {
    const payload = { ...bill, billStatus: 'Hoàn thành' };
    delete payload.details; 
    await api.put(`/bills/${bill.billId}`, payload);
    fetchPendingOrders();
  } catch (err) { alert("Lỗi cập nhật!"); }
};

// Shipper (Giao hàng) -> Bấm Đang giao hàng
const markAsShipped = async (bill) => {
  try {
    const payload = { ...bill, billStatus: 'Đang giao hàng' };
    delete payload.details; 
    await api.put(`/bills/${bill.billId}`, payload);
    fetchPendingOrders();
  } catch (err) { alert("Lỗi cập nhật!"); }
};

const formatTime = (dateStr) => {
  if (!dateStr) return '';
  return new Date(dateStr).toLocaleTimeString('vi-VN', { hour: '2-digit', minute: '2-digit' });
};
</script>

<style scoped>
.pager-alert {
  background: #f4f6f8; border: 1px dashed #bdc3c7; padding: 8px 10px; 
  border-radius: 6px; text-align: center; margin-bottom: 10px; color: #2c3e50; font-size: 15px;
}
.pager-alert b { font-size: 18px; color: #c0392b; }

.pager-alert.is-shipper {
  background: #fdf2e9; border-color: #e67e22; color: #d35400;
  animation: pulse 2s infinite; 
}
@keyframes pulse {
  0% { box-shadow: 0 0 0 0 rgba(230, 126, 34, 0.4); }
  70% { box-shadow: 0 0 0 6px rgba(230, 126, 34, 0); }
  100% { box-shadow: 0 0 0 0 rgba(230, 126, 34, 0); }
}
.small-alert { font-size: 13px; padding: 5px; }
.small-alert b { font-size: 15px; }
.badge.delivery { background: #e67e22; color: white; }

.kds-page { padding: 25px; min-height: 100vh; background: #2c3e50; }
.header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px; border-bottom: 2px solid #34495e; padding-bottom: 15px; }
.header h2 { margin: 0; color: #f1c40f; font-size: 28px; text-transform: uppercase; letter-spacing: 1px;}
.header-right { display: flex; align-items: center; gap: 15px; }
.timer-text { color: #bdc3c7; font-size: 14px; font-style: italic;}
.btn-refresh { background: #34495e; color: white; border: 1px solid #7f8c8d; padding: 8px 15px; border-radius: 5px; cursor: pointer; font-weight: bold; transition: 0.2s;}
.btn-refresh:hover { background: #7f8c8d; }
.btn-return-admin { background: #e74c3c; color: white; border: none; padding: 8px 15px; border-radius: 6px; cursor: pointer; font-weight: bold; transition: 0.2s; font-size: 16px;}
.btn-return-admin:hover { background: #c0392b; transform: translateX(-2px); }
.kds-container { display: flex; gap: 20px; align-items: flex-start; }
.kds-column { flex: 7; background: rgba(0,0,0,0.2); padding: 20px; border-radius: 10px; min-height: 500px;}
.column-ready { flex: 3; background: rgba(39, 174, 96, 0.1); border: 2px dashed #27ae60;}
.col-title { margin-top: 0; margin-bottom: 20px; font-size: 18px; border-bottom: 1px solid rgba(255,255,255,0.1); padding-bottom: 10px;}
.text-orange { color: #e67e22; }
.text-green { color: #2ecc71; }
.empty-state { text-align: center; color: #ecf0f1; margin-top: 50px; opacity: 0.6; }
.empty-state h2 { font-size: 24px; margin-bottom: 10px; color: #2ecc71;}
.kds-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(280px, 1fr)); gap: 20px; align-items: start;}
.ready-grid { grid-template-columns: 1fr; } 
.ticket { background: #fffcf2; padding: 20px; border-radius: 8px; box-shadow: 0 5px 15px rgba(0,0,0,0.3); position: relative;}
.ticket-pending { border-top: 8px solid #e67e22; }
.ticket-ready { border-left: 8px solid #27ae60; padding: 15px; background: #f0fdf4;}
.ticket::before { content: ''; position: absolute; top: 0; left: 15px; width: 30px; height: 10px; background: rgba(0,0,0,0.1); border-radius: 0 0 5px 5px;}
.ticket-ready::before { display: none; }
.head { display: flex; justify-content: space-between; align-items: center; border-bottom: 2px dashed #bdc3c7; padding-bottom: 10px; margin-bottom: 10px;}
.code { font-weight: 900; font-size: 22px; color: #2c3e50; }
.badge { font-size: 13px; font-weight: bold; padding: 4px 10px; border-radius: 4px; }
.badge.in-store { background: #2980b9; color: white; }
.badge.take-away { background: #8e44ad; color: white; }
.time { font-size: 14px; color: #7f8c8d; font-weight: bold; margin-bottom: 15px; text-align: right;}
.items-list { list-style: none; padding: 0; margin: 0 0 20px 0; min-height: 100px; }
.items-list li { padding: 8px 0; border-bottom: 1px solid #eee; }
.items-list li:last-child { border-bottom: none; }
.qty { font-size: 18px; color: #c0392b; margin-right: 8px; }
.drink-name { font-size: 16px; font-weight: bold; color: #2c3e50; }
.toppings { font-size: 13px; color: #e67e22; margin-top: 5px; font-weight: bold; padding-left: 30px;}
.btn-done { width: 100%; padding: 15px; background: #e67e22; color: white; border: none; font-weight: bold; font-size: 15px; border-radius: 5px; cursor: pointer; transition: 0.2s;}
.btn-done:hover { background: #d35400; transform: translateY(-2px); }

/* Nút bấm giao hàng - THÊM VÀO ĐÂY */
.btn-finish { width: 100%; padding: 10px; background: #27ae60; color: white; border: none; font-weight: bold; font-size: 14px; border-radius: 5px; cursor: pointer; transition: 0.2s; margin-top: 10px;}
.btn-finish:hover { background: #2ecc71; }
.btn-finish.delivery-btn { background: #e67e22; }
.btn-finish.delivery-btn:hover { background: #d35400; }
</style>