<template>
  <div class="cart-wrap">
    <div class="cart-header">
      <h2>🛒 Giỏ Hàng Của Bạn</h2>
      <button class="btn-back" @click="$router.push('/')">Tiếp tục mua hàng</button>
    </div>
    
    <div v-if="cart.length === 0" class="empty-cart">
      <div class="empty-icon">🥤</div>
      <h3>Giỏ hàng đang trống</h3>
      <p>Có vẻ như bạn chưa chọn thức uống nào. Hãy quay lại menu để đặt món nhé!</p>
      <button class="btn-primary" @click="$router.push('/')">Xem Menu</button>
    </div>

    <div v-else class="content">
      <div class="items">
        <div class="item-card" v-for="(item, idx) in cart" :key="idx">
          <div class="item-img">
            <img :src="item.image || 'https://via.placeholder.com/80?text=Drink'" alt="Drink" />
          </div>
          <div class="item-info">
            <h4 class="item-name">{{ item.drinkName }}</h4>
            <div class="item-toppings" v-if="item.toppings && item.toppings.length > 0">
              <span v-for="t in item.toppings" :key="t.toppingId">+ {{ t.toppingName }}</span>
            </div>
            <p class="item-note" v-if="item.note">📝 {{ item.note }}</p>
            <div class="item-price">{{ formatPrice(getItemPrice(item)) }}</div>
          </div>
          
          <div class="item-actions">
            <div class="qty-control">
              <button @click="updateQty(idx, -1)" :disabled="item.quantity <= 1">-</button>
              <input type="number" readonly :value="item.quantity || 1" />
              <button @click="updateQty(idx, 1)">+</button>
            </div>
            <button class="btn-remove" @click="remove(idx)">🗑️</button>
          </div>
        </div>
      </div>

      <div class="checkout-box">
        <h3>Tóm Tắt Đơn Hàng</h3>

        <div class="summary-lines">
          <div class="line">
            <span>Tạm tính ({{ cart.length }} món):</span>
            <span>{{ formatPrice(subTotal) }}</span>
          </div>
          <div class="line discount" v-if="appliedVoucher">
            <span>Khuyến mãi:</span>
            <span>- {{ formatPrice(appliedVoucher.discountAmount) }}</span>
          </div>
          <div class="line total">
            <span>Thành tiền:</span>
            <span class="total-price">{{ formatPrice(finalTotal) }}</span>
          </div>
        </div>

        <div v-if="!currentUser" class="login-prompt">
          <p>Vui lòng đăng nhập để nhập địa chỉ giao hàng và tiến hành thanh toán nhé!</p>
          <button class="btn-login-checkout" @click="goToLogin">
            👤 ĐĂNG NHẬP ĐỂ ĐẶT HÀNG
          </button>
        </div>

        <div v-else class="checkout-form">
          <div class="form-group address-box">
            <label>📍 Địa chỉ nhận hàng <span style="color:red">*</span></label>
            <textarea 
              v-model="deliveryAddress" 
              rows="3" 
              placeholder="Nhập số nhà, tên đường, phường/xã..."
              :class="{'error-border': addressError}"
            ></textarea>
            <small v-if="addressError" style="color: red; font-weight: bold; margin-top: 5px; display: block;">Vui lòng nhập địa chỉ giao hàng!</small>
          </div>

          <div class="form-group">
            <label>💳 Phương thức thanh toán</label>
            <select v-model="paymentMethod">
              <option value="Tiền mặt">💵 Tiền mặt (Thanh toán cho Shipper)</option>
              <option value="Chuyển khoản">📱 Chuyển khoản ngân hàng trực tuyến</option>
            </select>
          </div>

          <div class="voucher-box">
            <label>Mã Giảm Giá</label>
            <div class="voucher-input-group">
              <input type="text" v-model="voucherCode" placeholder="Nhập mã..." :disabled="appliedVoucher" style="text-transform: uppercase;" />
              <button v-if="!appliedVoucher" @click="applyVoucher" class="btn-apply">Áp dụng</button>
              <button v-else @click="removeVoucher" class="btn-remove-voucher">Hủy</button>
            </div>
            <small class="voucher-success" v-if="appliedVoucher">
              ✔️ Đã áp dụng mã giảm {{ formatPrice(appliedVoucher.discountAmount) }}
            </small>
          </div>

          <button class="btn-order" @click="placeOrder" :disabled="loading">
            {{ loading ? 'Đang xử lý...' : '🚀 ĐẶT HÀNG NGAY' }}
          </button>
        </div>
      </div>
    </div>

    <!-- MODAL THANH TOÁN ONLINE (QR CODE) -->
    <div class="modal-overlay" v-if="showQRModal">
      <div class="qr-modal-content">
        <div class="success-icon">✅</div>
        <h3>Đặt Hàng Thành Công!</h3>
        <p class="subtitle">Mã đơn của bạn: <strong class="highlight-code">{{ createdBillInfo?.billCode }}</strong></p>

        <div class="qr-instruction">
          Quét mã QR dưới đây để thanh toán cho quán nhé:
        </div>

        <div class="qr-box">
          <img :src="`https://img.vietqr.io/image/MB-123456789-compact2.png?amount=${createdBillInfo?.totalPrice}&addInfo=${createdBillInfo?.billCode}`" alt="QR Code" class="qr-image" />
        </div>

        <h2 class="qr-total">{{ formatPrice(createdBillInfo?.totalPrice) }}</h2>
        <p class="qr-note">Đơn hàng đang ở trạng thái <b>"Chờ xác nhận"</b>. Nhân viên sẽ xử lý ngay sau khi đối chiếu giao dịch.</p>

        <div class="modal-actions">
          <button class="btn-primary full-width" @click="finishOnlinePayment">
            Tiếp Tục & Xem Trạng Thái Đơn
          </button>
        </div>
      </div>
    </div>

  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue';
import { useRouter } from 'vue-router';
import api from '../../services/api';

const router = useRouter();
const cart = ref([]);
const loading = ref(false);

// Thông tin đặt hàng
const paymentMethod = ref('Tiền mặt');
const deliveryAddress = ref('');
const addressError = ref(false);

// Xử lý Voucher
const voucherCode = ref('');
const appliedVoucher = ref(null);

// Lấy thông tin user đăng nhập
const currentUser = ref(null);

// Xử lý Hóa đơn Online (QR)
const showQRModal = ref(false);
const createdBillInfo = ref(null);

onMounted(() => { 
  const userStr = localStorage.getItem('user');
  if (userStr) {
    currentUser.value = JSON.parse(userStr);
    if (currentUser.value.address) {
      deliveryAddress.value = currentUser.value.address;
    }
  }

  cart.value = JSON.parse(localStorage.getItem('cart')) || []; 
  cart.value.forEach(item => { if (!item.quantity) item.quantity = 1; });
});

const getItemPrice = (item) => {
  let base = item.drinkPrice || 0;
  let topPrice = (item.toppings || []).reduce((sum, t) => sum + (t.price || t.toppingPrice || 0), 0);
  return base + topPrice;
};

const subTotal = computed(() => {
  return cart.value.reduce((sum, item) => sum + (getItemPrice(item) * item.quantity), 0);
});

const finalTotal = computed(() => {
  let total = subTotal.value;
  if (appliedVoucher.value && appliedVoucher.value.discountAmount) {
    total -= appliedVoucher.value.discountAmount;
  }
  return total > 0 ? total : 0;
});

const updateQty = (idx, delta) => {
  if (cart.value[idx].quantity + delta > 0) {
    cart.value[idx].quantity += delta;
    saveCart();
  }
};

const remove = (idx) => { 
  if(confirm('Bạn muốn bỏ món này khỏi giỏ?')) {
    cart.value.splice(idx, 1); 
    saveCart(); 
  }
};

const saveCart = () => { 
  localStorage.setItem('cart', JSON.stringify(cart.value)); 
  window.dispatchEvent(new Event('cart-updated')); 
  
  if (appliedVoucher.value && subTotal.value < appliedVoucher.value.minOrderValue) {
    alert("Tổng đơn hàng không còn đủ điều kiện áp dụng mã giảm giá này. Hệ thống tự động gỡ mã!");
    removeVoucher();
  }
};

const applyVoucher = async () => {
  if (!voucherCode.value) return;
  try {
    const res = await api.post('/vouchers/check', { 
      code: voucherCode.value.toUpperCase(), 
      orderTotal: subTotal.value 
    });
    
    if (res.data.status === 'success') {
      appliedVoucher.value = res.data.data;
    } else {
      alert(res.data.message);
    }
  } catch (err) {
    alert(err.response?.data?.message || "Mã giảm giá không hợp lệ hoặc chưa đủ điều kiện!");
  }
};

const removeVoucher = () => {
  appliedVoucher.value = null;
  voucherCode.value = '';
};

const goToLogin = () => {
  router.push('/login');
};

// ĐẶT HÀNG
const placeOrder = async () => {
  if (!currentUser.value) {
    alert("Vui lòng đăng nhập để có thể đặt hàng!");
    router.push('/login');
    return;
  }

  if (!deliveryAddress.value || deliveryAddress.value.trim() === '') {
    addressError.value = true;
    return;
  }
  addressError.value = false;

  if (cart.value.length === 0) return;
  loading.value = true;
  
  try {
    const billPayload = {
      orderType: 'Giao hàng',
      paymentMethod: paymentMethod.value,
      totalPrice: finalTotal.value,
      billStatus: 'Chờ xác nhận', 
      deliveryAddress: deliveryAddress.value,
      user: { userId: currentUser.value.userId },
      voucher: appliedVoucher.value ? { voucherId: appliedVoucher.value.voucherId } : null
    };

    const resBill = await api.post('/bills', billPayload);
    const createdBill = resBill.data.data;

    for (const item of cart.value) {
      const detailPayload = {
        bill: { billId: createdBill.billId },
        drink: { drinkId: item.drinkId },
        bdQuantity: item.quantity,
        unitPrice: item.drinkPrice,
        note: item.note || '',
        bdStatus: 'Chờ xác nhận'
      };
      
      const resDetail = await api.post('/bill-details', detailPayload);
      const createdDetail = resDetail.data.data;

      if (item.toppings && item.toppings.length > 0) {
        for (const top of item.toppings) {
          await api.post('/bill-detail-toppings', {
            billDetail: { bdId: createdDetail.bdId },
            topping: { toppingId: top.toppingId || top.id },
            bdtPrice: top.toppingPrice || top.price
          });
        }
      }
    }

    // ================== ĐÃ SỬA CHỖ NÀY ==================
    // Gán thông tin cho Modal QR TRƯỚC KHI xóa giỏ hàng
    if (paymentMethod.value === 'Chuyển khoản') {
      createdBillInfo.value = {
        billCode: createdBill.billCode,
        totalPrice: finalTotal.value // Lấy tổng tiền đang tính toán
      };
      showQRModal.value = true;
    } else {
      alert("🎉 Đặt hàng thành công! Mã đơn của bạn là: " + createdBill.billCode);
      router.push('/customer/orders'); 
    }

    // Xóa giỏ hàng sau khi đã xử lý xong logic hiển thị
    cart.value = [];
    localStorage.removeItem('cart');
    window.dispatchEvent(new Event('cart-updated')); 
    // ====================================================

  } catch (err) {
    alert("Có lỗi xảy ra khi đặt hàng! " + (err.response?.data?.message || ""));
  } finally {
    loading.value = false;
  }
};

// Đóng modal QR và chuyển trang
const finishOnlinePayment = () => {
  showQRModal.value = false;
  router.push('/customer/orders');
};

const formatPrice = (p) => new Intl.NumberFormat('vi-VN', { style: 'currency', currency: 'VND' }).format(p || 0);
</script>

<style scoped>
/* CSS cũ giữ nguyên */
.cart-wrap { max-width: 1000px; margin: 40px auto; padding: 0 20px;}
.cart-header { display: flex; justify-content: space-between; align-items: center; border-bottom: 2px solid #eee; padding-bottom: 15px; margin-bottom: 25px;}
.cart-header h2 { margin: 0; color: #2c3e50; }
.btn-back { background: #ecf0f1; border: none; padding: 10px 15px; border-radius: 5px; cursor: pointer; font-weight: bold; color: #34495e;}
.btn-back:hover { background: #bdc3c7; }

.empty-cart { text-align: center; padding: 50px 0; background: white; border-radius: 10px; box-shadow: 0 5px 15px rgba(0,0,0,0.05);}
.empty-icon { font-size: 60px; margin-bottom: 15px; }
.empty-cart h3 { color: #2c3e50; margin: 0 0 10px 0; }
.empty-cart p { color: #7f8c8d; margin-bottom: 20px; }
.btn-primary { background: #2980b9; color: white; padding: 12px 25px; border: none; border-radius: 25px; font-weight: bold; cursor: pointer; transition: 0.3s;}
.btn-primary:hover { background: #3498db; }

.content { display: flex; gap: 30px; align-items: flex-start; }

.items { flex: 6.5; display: flex; flex-direction: column; gap: 15px;}
.item-card { display: flex; align-items: center; background: white; padding: 15px; border-radius: 10px; box-shadow: 0 2px 8px rgba(0,0,0,0.05); gap: 15px;}
.item-img img { width: 80px; height: 80px; border-radius: 8px; object-fit: cover;}
.item-info { flex: 1; }
.item-name { margin: 0 0 5px 0; color: #2c3e50; font-size: 18px;}
.item-toppings { font-size: 12px; color: #7f8c8d; margin-bottom: 5px;}
.item-toppings span { background: #ecf0f1; padding: 2px 8px; border-radius: 12px; margin-right: 5px; display: inline-block;}
.item-note { font-size: 13px; color: #e67e22; font-style: italic; margin: 0 0 5px 0;}
.item-price { font-weight: bold; color: #d35400; font-size: 16px;}

.item-actions { display: flex; flex-direction: column; align-items: flex-end; gap: 15px;}
.qty-control { display: flex; align-items: center; border: 1px solid #ccc; border-radius: 5px; overflow: hidden;}
.qty-control button { background: #f8f9fa; border: none; width: 30px; height: 30px; cursor: pointer; font-weight: bold; font-size: 16px; color: #2c3e50;}
.qty-control button:hover:not(:disabled) { background: #e2e6ea; }
.qty-control button:disabled { opacity: 0.5; cursor: not-allowed; }
.qty-control input { width: 40px; text-align: center; border: none; border-left: 1px solid #ccc; border-right: 1px solid #ccc; font-weight: bold; pointer-events: none;}
.btn-remove { background: none; border: none; cursor: pointer; font-size: 18px; color: #e74c3c; padding: 5px; transition: 0.2s;}
.btn-remove:hover { transform: scale(1.2); }

.checkout-box { flex: 3.5; background: white; padding: 25px; border-radius: 10px; box-shadow: 0 5px 15px rgba(0,0,0,0.08); position: sticky; top: 20px;}
.checkout-box h3 { margin: 0 0 20px 0; color: #2c3e50; border-bottom: 2px solid #f4f6f8; padding-bottom: 10px;}

.form-group { margin-bottom: 15px; }
.form-group label { display: block; font-size: 13px; font-weight: bold; color: #7f8c8d; margin-bottom: 5px;}
.form-group select, .form-group textarea { width: 100%; padding: 10px; border: 1px solid #bdc3c7; border-radius: 5px; font-family: inherit; outline: none; box-sizing: border-box;}
.form-group textarea:focus { border-color: #3498db; }

.error-border { border-color: #e74c3c !important; background: #fdedec; }
.address-box { animation: fadeIn 0.3s ease; }
@keyframes fadeIn { from { opacity: 0; transform: translateY(-5px); } to { opacity: 1; transform: translateY(0); } }

.voucher-box { margin-bottom: 20px; background: #fdfaf4; padding: 15px; border-radius: 8px; border: 1px dashed #f1c40f;}
.voucher-box label { display: block; font-size: 13px; font-weight: bold; color: #d35400; margin-bottom: 8px;}
.voucher-input-group { display: flex; gap: 10px; }
.voucher-input-group input { flex: 1; padding: 10px; border: 1px solid #bdc3c7; border-radius: 5px; text-transform: uppercase;}
.btn-apply { background: #2c3e50; color: white; border: none; padding: 0 15px; border-radius: 5px; font-weight: bold; cursor: pointer;}
.btn-remove-voucher { background: #e74c3c; color: white; border: none; padding: 0 15px; border-radius: 5px; font-weight: bold; cursor: pointer;}
.voucher-success { display: block; color: #27ae60; font-size: 13px; margin-top: 8px; font-weight: bold;}

.summary-lines { border-top: 1px solid #eee; margin-top: 20px; padding-top: 20px; margin-bottom: 15px;}
.line { display: flex; justify-content: space-between; margin-bottom: 10px; color: #34495e; font-size: 15px;}
.line.discount { color: #27ae60; font-weight: bold; }
.line.total { border-top: 2px dashed #ccc; padding-top: 15px; font-size: 18px; font-weight: bold; margin-top: 15px;}
.total-price { color: #c0392b; font-size: 24px;}

.login-prompt { margin-top: 20px; text-align: center; padding: 20px; background: #fef9e7; border-radius: 8px; border: 1px dashed #f39c12; animation: fadeIn 0.3s ease;}
.login-prompt p { margin: 0 0 15px 0; color: #d35400; font-size: 14px; font-weight: bold;}
.btn-login-checkout { background: #f39c12; color: white; padding: 15px; width: 100%; border: none; border-radius: 8px; font-weight: bold; font-size: 16px; cursor: pointer; transition: 0.3s; box-shadow: 0 4px 6px rgba(243, 156, 18, 0.2);}
.btn-login-checkout:hover { background: #e67e22; transform: translateY(-2px);}

.checkout-form { animation: fadeIn 0.3s ease; }

.btn-order { background: #27ae60; color: white; padding: 15px; width: 100%; border: none; border-radius: 8px; font-weight: bold; font-size: 16px; cursor: pointer; transition: 0.3s; box-shadow: 0 4px 6px rgba(39,174,96,0.2);}
.btn-order:hover:not(:disabled) { background: #2ecc71; transform: translateY(-2px); box-shadow: 0 6px 10px rgba(39,174,96,0.3);}
.btn-order:disabled { background: #95a5a6; cursor: not-allowed; box-shadow: none;}

@media (max-width: 768px) {
  .content { flex-direction: column; }
  .items, .checkout-box { flex: 1; width: 100%; }
}

/* CSS MỚI CHO MODAL QR CODE */
.modal-overlay { position: fixed; top: 0; left: 0; width: 100vw; height: 100vh; background: rgba(0,0,0,0.6); display: flex; justify-content: center; align-items: center; z-index: 2000; }
.qr-modal-content { background: white; padding: 35px 30px; border-radius: 15px; width: 400px; max-width: 90%; box-shadow: 0 10px 30px rgba(0,0,0,0.3); text-align: center; animation: fadeIn 0.3s ease;}
.success-icon { font-size: 50px; margin-bottom: 10px; }
.qr-modal-content h3 { margin: 0 0 5px 0; color: #27ae60; font-size: 24px; }
.subtitle { color: #7f8c8d; font-size: 15px; margin-bottom: 20px;}
.highlight-code { color: #8e44ad; font-size: 18px; }

.qr-instruction { background: #fef9e7; color: #d35400; padding: 10px; border-radius: 5px; font-size: 14px; font-weight: bold; margin-bottom: 15px;}
.qr-box { background: #f4f6f8; padding: 15px; border-radius: 10px; display: flex; justify-content: center; margin-bottom: 15px; border: 1px dashed #bdc3c7;}
.qr-image { width: 220px; height: 220px; mix-blend-mode: multiply; }
.qr-total { color: #c0392b; margin: 0 0 10px 0; font-size: 28px;}
.qr-note { font-size: 13px; color: #7f8c8d; line-height: 1.4; margin-bottom: 20px;}

.full-width { width: 100%; padding: 15px; font-size: 16px; border-radius: 8px;}
</style>