<template>
  <div class="orders-page">
    <div class="header-section">
      <h2>📦 Lịch Sử Đơn Hàng</h2>
      <p>Theo dõi trạng thái các thức uống bạn đã đặt tại PolyCafe.</p>
    </div>

    <div v-if="loading" class="loading-state">
      ⏳ Đang tải thông tin đơn hàng...
    </div>

    <div v-else-if="orders.length === 0" class="empty-state">
      <div class="empty-icon">📝</div>
      <h3>Bạn chưa có đơn hàng nào</h3>
      <p>Hãy đặt thử một ly nước yêu thích và quay lại đây nhé!</p>
      <router-link to="/" class="btn-primary">Xem Menu Quán</router-link>
    </div>

    <div v-else class="orders-list">
      <div class="order-card" v-for="order in orders" :key="order.billId">
        
        <div class="order-header">
          <div class="order-code">
            Mã đơn: <strong>{{ order.billCode }}</strong>
            <span class="order-date">{{ formatDate(order.createdAt) }}</span>
          </div>
          <div class="order-status" :class="getStatusClass(order.billStatus)">
            {{ order.billStatus }}
          </div>
        </div>

        <div class="order-body">
          <div class="info-row">
            <span>Loại đơn:</span>
            <strong>{{ order.orderType }}</strong>
          </div>
          <div class="info-row" v-if="order.orderType === 'Mang về' || order.orderType === 'Giao hàng'">
            <span>Giao đến:</span>
            <span class="address-text">{{ order.deliveryAddress }}</span>
          </div>
          <div class="info-row">
            <span>Thanh toán:</span>
            <span>{{ order.paymentMethod }}</span>
          </div>
          <div class="info-row total-row">
            <span>Tổng tiền:</span>
            <span class="price">{{ formatPrice(order.totalPrice) }}</span>
          </div>
        </div>

        <div class="order-footer">
          <button class="btn-view" @click="viewDetails(order)">Chi tiết món</button>
          
          <div class="action-buttons">
            <button 
              v-if="order.billStatus === 'Chờ xác nhận'" 
              class="btn-cancel" 
              @click="cancelOrder(order)"
            >
              Hủy đơn
            </button>

            <button 
              v-if="order.billStatus === 'Đang giao hàng' || order.billStatus === 'Chờ nhận hàng'" 
              class="btn-receive" 
              @click="confirmReceipt(order)"
            >
              ✅ Đã nhận được hàng
            </button>
          </div>
        </div>
        
      </div>
    </div>

    <div class="modal-overlay" v-if="showModal">
      <div class="modal-content">
        <div class="modal-header">
          <h3>Chi tiết đơn <span class="highlight">{{ selectedOrder?.billCode }}</span></h3>
          <button class="btn-close" @click="showModal = false">❌</button>
        </div>
        
        <div class="modal-body">
          <div v-if="loadingDetails" class="text-center" style="padding: 20px;">Đang tải...</div>
          <div class="item-list" v-else>
            <div class="item-row" v-for="detail in orderDetails" :key="detail.bdId">
              <div class="item-main">
                <b class="item-name">{{ detail.drink?.drinkName }}</b>
                <div class="item-toppings" v-if="detail.toppings && detail.toppings.length">
                  + {{ detail.toppings.map(t => t.topping?.toppingName || t.toppingName).join(', ') }}
                </div>
                <div class="item-note" v-if="detail.note">📝 {{ detail.note }}</div>
              </div>
              <div class="item-qty">x{{ detail.bdQuantity || detail.quantity }}</div>
              <div class="item-price">{{ formatPrice(detail.unitPrice * (detail.bdQuantity || detail.quantity)) }}</div>
            </div>
          </div>
        </div>

        <div class="modal-footer">
          <div class="summary-total">
            Tổng thanh toán: <strong>{{ formatPrice(selectedOrder?.totalPrice) }}</strong>
          </div>
        </div>
      </div>
    </div>

  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue';
import { useRouter } from 'vue-router';
import api from '../../services/api';

const router = useRouter();
const orders = ref([]);
const loading = ref(true);

const currentUser = ref(null);

// Modal state
const showModal = ref(false);
const selectedOrder = ref(null);
const orderDetails = ref([]);
const loadingDetails = ref(false);

onMounted(() => {
  const userStr = localStorage.getItem('user');
  if (!userStr) {
    alert("Vui lòng đăng nhập để xem đơn hàng!");
    router.push('/login');
    return;
  }
  currentUser.value = JSON.parse(userStr);
  fetchMyOrders();
});

const fetchMyOrders = async () => {
  loading.value = true;
  try {
    // Gọi API lấy lịch sử đơn hàng của User này
    const res = await api.get(`/bills/user/${currentUser.value.userId}`);
    // Đảo ngược mảng để đơn mới nhất lên đầu
    orders.value = (res.data.data || []).reverse();
  } catch (err) {
    console.error("Lỗi lấy đơn hàng:", err);
  } finally {
    loading.value = false;
  }
};

// Khách tự tay chốt đơn
const confirmReceipt = async (order) => {
  if (confirm("Xác nhận bạn đã nhận được nước và thanh toán đầy đủ?")) {
    try {
      await api.put(`/bills/${order.billId}/status`, null, {
        params: { status: 'Hoàn thành' }
      });
      alert("Cảm ơn bạn đã ủng hộ PolyCafe! ❤️");
      fetchMyOrders(); // Tải lại danh sách
    } catch (err) {
      alert("Có lỗi xảy ra, vui lòng thử lại!");
    }
  }
};

// Khách tự hủy đơn
const cancelOrder = async (order) => {
  const reason = prompt("Vui lòng cho quán biết lý do bạn hủy đơn nhé (để quán cải thiện tốt hơn):");
  if (reason === null) return; // Khách bấm Cancel popup
  
  try {
    await api.put(`/bills/${order.billId}/status`, null, {
      params: { 
        status: 'Đã hủy',
        cancelReason: reason || 'Khách đổi ý'
      }
    });
    alert("Hủy đơn thành công!");
    fetchMyOrders();
  } catch (err) {
    alert("Không thể hủy đơn lúc này!");
  }
};

// Xem chi tiết món (Giống hệt luồng Admin)
const viewDetails = async (order) => {
  selectedOrder.value = order;
  showModal.value = true;
  orderDetails.value = [];
  loadingDetails.value = true;

  try {
    const res = await api.get(`/bill-details/bill/${order.billId}`);
    const detailsData = res.data.data || [];
    
    for (let detail of detailsData) {
      try {
         const topRes = await api.get(`/bill-detail-toppings/detail/${detail.bdId}`);
         detail.toppings = topRes.data.data || [];
      } catch (e) {
         detail.toppings = [];
      }
    }
    orderDetails.value = detailsData;
  } catch (err) {
    console.error("Lỗi lấy chi tiết:", err);
  } finally {
    loadingDetails.value = false;
  }
};

const formatPrice = (price) => new Intl.NumberFormat('vi-VN', { style: 'currency', currency: 'VND' }).format(price || 0);

const formatDate = (dateStr) => {
  if (!dateStr) return '';
  const d = new Date(dateStr);
  return d.toLocaleString('vi-VN', { 
    day: '2-digit', month: '2-digit', year: 'numeric',
    hour: '2-digit', minute: '2-digit'
  });
};

const getStatusClass = (status) => {
  if (status === 'Đã hủy') return 'status-danger';
  if (status === 'Chờ xác nhận') return 'status-secondary';
  if (status === 'Chờ pha chế') return 'status-warning';
  if (status === 'Đang giao hàng' || status === 'Chờ nhận hàng' || status === 'Chờ khách lấy') return 'status-info';
  if (status === 'Hoàn thành') return 'status-success';
  return 'status-secondary';
};
</script>

<style scoped>
.orders-page { max-width: 900px; margin: 40px auto; padding: 0 20px; min-height: 60vh;}
.header-section { margin-bottom: 30px; border-bottom: 2px solid #eee; padding-bottom: 15px;}
.header-section h2 { margin: 0 0 5px 0; color: #2c3e50; font-size: 26px;}
.header-section p { margin: 0; color: #7f8c8d; font-size: 15px;}

.loading-state, .empty-state { text-align: center; padding: 50px 0; background: white; border-radius: 12px; box-shadow: 0 4px 15px rgba(0,0,0,0.05);}
.empty-icon { font-size: 50px; margin-bottom: 15px;}
.empty-state h3 { color: #2c3e50; margin-bottom: 10px;}
.empty-state p { color: #7f8c8d; margin-bottom: 20px;}
.btn-primary { display: inline-block; background: #2980b9; color: white; padding: 10px 20px; border-radius: 20px; text-decoration: none; font-weight: bold; transition: 0.3s;}
.btn-primary:hover { background: #3498db;}

.orders-list { display: flex; flex-direction: column; gap: 20px;}
.order-card { background: white; border-radius: 12px; box-shadow: 0 4px 12px rgba(0,0,0,0.05); overflow: hidden; border: 1px solid #f4f6f8;}

.order-header { display: flex; justify-content: space-between; align-items: center; padding: 15px 20px; background: #fdfaf4; border-bottom: 1px dashed #eee;}
.order-code { display: flex; flex-direction: column; color: #34495e;}
.order-date { font-size: 13px; color: #95a5a6; margin-top: 3px;}

.order-status { padding: 5px 12px; border-radius: 20px; font-size: 13px; font-weight: bold; border: 1px solid transparent;}
.status-success { background: #eafaf1; color: #27ae60; border-color: #27ae60;}
.status-warning { background: #fef9e7; color: #f39c12; border-color: #f39c12;}
.status-danger { background: #fdedec; color: #e74c3c; border-color: #e74c3c;}
.status-info { background: #ebf5fb; color: #2980b9; border-color: #2980b9;}
.status-secondary { background: #f4f6f8; color: #7f8c8d; border-color: #bdc3c7;}

.order-body { padding: 15px 20px;}
.info-row { display: flex; justify-content: space-between; margin-bottom: 10px; font-size: 14px; color: #34495e;}
.address-text { text-align: right; max-width: 60%; color: #7f8c8d; font-style: italic;}
.total-row { border-top: 1px solid #eee; padding-top: 10px; margin-top: 10px; font-size: 16px; font-weight: bold;}
.price { color: #c0392b; font-size: 18px;}

.order-footer { display: flex; justify-content: space-between; align-items: center; padding: 15px 20px; background: #fafbfc; border-top: 1px solid #f4f6f8;}
.btn-view { background: transparent; border: 1px solid #bdc3c7; color: #34495e; padding: 8px 15px; border-radius: 6px; font-weight: bold; cursor: pointer; transition: 0.2s;}
.btn-view:hover { background: #ecf0f1;}

.action-buttons { display: flex; gap: 10px;}
.btn-cancel { background: white; border: 1px solid #e74c3c; color: #e74c3c; padding: 8px 15px; border-radius: 6px; font-weight: bold; cursor: pointer; transition: 0.2s;}
.btn-cancel:hover { background: #fdedec;}

.btn-receive { background: #27ae60; border: none; color: white; padding: 8px 15px; border-radius: 6px; font-weight: bold; cursor: pointer; transition: 0.2s; box-shadow: 0 2px 5px rgba(39, 174, 96, 0.3);}
.btn-receive:hover { background: #2ecc71; transform: translateY(-1px);}

/* MODAL CHI TIẾT */
.modal-overlay { position: fixed; top: 0; left: 0; width: 100vw; height: 100vh; background: rgba(0,0,0,0.5); display: flex; justify-content: center; align-items: center; z-index: 1000;}
.modal-content { background: white; width: 500px; max-width: 95%; border-radius: 12px; overflow: hidden; box-shadow: 0 10px 30px rgba(0,0,0,0.2); max-height: 85vh; display: flex; flex-direction: column;}
.modal-header { display: flex; justify-content: space-between; align-items: center; padding: 15px 20px; background: #fdfaf4; border-bottom: 1px solid #eee;}
.modal-header h3 { margin: 0; color: #2c3e50; font-size: 18px;}
.highlight { color: #e67e22;}
.btn-close { background: none; border: none; font-size: 18px; cursor: pointer;}

.modal-body { padding: 20px; overflow-y: auto;}
.item-row { display: flex; justify-content: space-between; align-items: flex-start; border-bottom: 1px dashed #eee; padding-bottom: 12px; margin-bottom: 12px;}
.item-row:last-child { border-bottom: none; margin-bottom: 0; padding-bottom: 0;}
.item-main { flex: 1; padding-right: 15px;}
.item-name { color: #2c3e50; font-size: 15px;}
.item-toppings { font-size: 12px; color: #7f8c8d; margin-top: 3px;}
.item-note { font-size: 12px; color: #e67e22; font-style: italic; margin-top: 3px;}
.item-qty { font-weight: bold; color: #7f8c8d; width: 30px; text-align: right;}
.item-price { font-weight: bold; color: #d35400; width: 80px; text-align: right;}

.modal-footer { padding: 15px 20px; background: #fafbfc; border-top: 1px solid #eee; text-align: right;}
.summary-total { font-size: 16px; color: #34495e;}
.summary-total strong { color: #c0392b; font-size: 20px; margin-left: 10px;}
</style>