<template>
  <div class="pos-layout">
    <div class="menu-col no-print">
      <div class="header-search">
        <div style="display: flex; align-items: center; gap: 15px;">
          <button v-if="currentUser && currentUser.role == 2" @click="$router.push('/admin')" class="btn-return-admin">
            🔙 Quản Trị
          </button>
          <h2>💻 PolyCafe POS</h2>
        </div>
        
        <div class="cat-filters">
          <button class="cat-btn" :class="{active: selectedCat === null}" @click="selectedCat = null">Tất cả</button>
          <button 
            class="cat-btn" 
            v-for="cat in categories" 
            :key="cat.categoryId" 
            :class="{active: selectedCat === cat.categoryId}" 
            @click="selectedCat = cat.categoryId"
          >
            {{ cat.categoryName }}
          </button>
        </div>

        <div class="search-box-wrap">
          <span>🔍</span>
          <input type="text" v-model="searchQuery" placeholder="Tìm tên đồ uống (VD: Cà phê)..." class="search-box" />
        </div>
      </div>
      
      <div class="drink-grid">
        <div v-if="filteredDrinks.length === 0" class="empty-menu">Không tìm thấy thức uống nào!</div>
        <div class="drink-card" v-for="d in filteredDrinks" :key="d.drinkId" @click="addToBill(d)">
          <img :src="d.image || 'https://images.unsplash.com/photo-1559525839-b184a4d698c7?w=500&q=80'" class="img-thumb" />
          <div class="info">
            <h4>{{ d.drinkName }}</h4>
            <p class="price">{{ formatPrice(d.drinkPrice) }}</p>
          </div>
        </div>
      </div>
    </div>

    <div class="bill-col no-print">
      <div class="bill-title-bar">
        <h3 class="bill-title">🧾 Hóa Đơn Hiện Tại</h3>
        <button class="btn-clear" v-if="currentBill.length > 0" @click="clearBill">🗑️ Hủy Bill</button>
      </div>
      
      <div class="bill-items">
        <div v-if="currentBill.length === 0" class="empty-msg">
          <div class="empty-icon">🛒</div>
          Chưa có món nào được chọn
        </div>
        
        <div class="bill-item" v-for="(item, index) in currentBill" :key="index">
          <div class="item-main">
            <div class="item-name-qty">
              <div class="mini-qty">
                <button @click="updateQty(index, -1)" :disabled="item.quantity <= 1">-</button>
                <span>{{ item.quantity }}</span>
                <button @click="updateQty(index, 1)">+</button>
              </div>
              <b>{{ item.drinkName }}</b>
            </div>
            <span class="item-price">{{ formatPrice(getItemTotal(item)) }}</span>
          </div>
          
          <div class="item-actions">
            <select v-model="item.selectedTopping" @change="addTopping(index)" class="topping-select">
              <option :value="null">-- Thêm Topping --</option>
              <option v-for="t in toppings" :key="t.toppingId" :value="t">
                + {{ t.toppingName }} ({{ formatPrice(t.toppingPrice) }})
              </option>
            </select>
            <input type="text" v-model="item.note" @change="savePosBill" placeholder="Ghi chú (Ít đá...)" class="note-input" />
            <button @click="removeItem(index)" class="btn-remove" title="Xóa món">❌</button>
          </div>
          
          <ul class="topping-list" v-if="item.toppings && item.toppings.length > 0">
            <li v-for="(top, tIdx) in item.toppings" :key="tIdx">
              <span>+ {{ top.toppingName }} ({{ formatPrice(top.toppingPrice) }})</span>
              <span class="remove-topping" @click="removeTopping(index, tIdx)">Hủy</span>
            </li>
          </ul>
        </div>
      </div>

      <div class="checkout-zone">
        <div class="pos-voucher">
          <input type="text" v-model="voucherCode" placeholder="Mã giảm giá..." :disabled="appliedVoucher" style="text-transform: uppercase;"/>
          <button v-if="!appliedVoucher" @click="applyVoucher" class="btn-sm-apply">Áp dụng</button>
          <button v-else @click="removeVoucher" class="btn-sm-remove">Hủy mã</button>
        </div>
        <div v-if="appliedVoucher" class="voucher-alert">✔️ Áp mã thành công: -{{ formatPrice(appliedVoucher.discountAmount) }}</div>

        <!-- THÊM Ô NHẬP THẺ RUNG -->
        <div class="summary-row type-row">
          <span>🔔 Thẻ Rung Số (Tùy chọn):</span>
          <input type="text" v-model="pagerCode" @change="savePosBill" placeholder="Để trống nếu không dùng" class="type-select" style="width: 170px; text-align: center;" />
        </div>

        <div class="summary-row type-row">
          <span>Loại đơn:</span>
          <select v-model="orderType" @change="savePosBill" class="type-select">
            <option value="Tại quán">🍽️ Tại quán</option>
            <option value="Mang đi">🛍️ Mang đi (Takeaway)</option>
          </select>
        </div>

        <div class="summary-row type-row">
          <span>Thanh toán:</span>
          <select v-model="paymentMethod" @change="savePosBill" class="type-select">
            <option value="Tiền mặt">💵 Tiền mặt</option>
            <option value="Chuyển khoản">💳 Chuyển khoản (Mã QR)</option>
          </select>
        </div>

        <div class="summary-row total-row">
          <span>TỔNG CỘNG:</span> 
          <b>{{ formatPrice(finalTotal) }}</b>
        </div>

        <button class="btn-pay" @click="handleCheckoutClick" :disabled="loading || currentBill.length === 0">
          {{ loading ? 'ĐANG XỬ LÝ...' : '💵 XÁC NHẬN THANH TOÁN' }}
        </button>
      </div>
    </div>

    <div class="modal-overlay no-print" v-if="showQRModal">
      <div class="qr-modal-content">
        <h3 style="text-align: center; color: #2c3e50; margin-top: 0;">📱 Mã QR Chuyển Khoản</h3>
        <p style="text-align: center; color: #7f8c8d; font-size: 14px;">Vui lòng yêu cầu khách hàng quét mã dưới đây</p>
        
        <div class="qr-box">
          <img :src="`https://img.vietqr.io/image/MB-123456789-compact2.png?amount=${finalTotal}&addInfo=Thanh toan PolyCafe`" alt="QR Code" class="qr-image" />
        </div>
        <h2 style="text-align: center; color: #c0392b; margin: 10px 0;">{{ formatPrice(finalTotal) }}</h2>

        <div class="modal-actions">
          <button class="btn-cancel" @click="showQRModal = false">Hủy Bỏ</button>
          <button class="btn-pay" @click="processCheckoutAPI" :disabled="loading">
            {{ loading ? 'ĐANG LƯU...' : '✅ ĐÃ NHẬN ĐƯỢC TIỀN' }}
          </button>
        </div>
      </div>
    </div>

    <div class="modal-overlay no-print" v-if="showReceiptModal">
      <div class="receipt-preview-wrap">
        <div class="receipt-actions no-print">
          <button class="btn-print" @click="printReceipt">🖨️ IN HÓA ĐƠN</button>
          <button class="btn-cancel-print" @click="closeReceiptAndReset">Đóng</button>
        </div>
        
        <div class="receipt-paper" id="print-area">
          <div class="receipt-header">
            <h2>POLYCAFE</h2>
            <p>Số 1 Trịnh Văn Bô, Nam Từ Liêm, HN</p>
            <p>Hotline: 0987.654.321</p>
            <p>WiFi: PolyCafe_Free - Pass: 12345678</p>
            <div class="dashed-line"></div>
            <h3>HÓA ĐƠN THANH TOÁN</h3>
            <p><b>Mã đơn:</b> {{ lastOrder?.billCode }}</p>
            <!-- THÊM DÒNG IN THẺ RUNG (NẾU CÓ) -->
            <p v-if="lastOrder?.pagerCode"><b>Thẻ Rung:</b> <span style="font-size: 16px; font-weight: bold;">{{ lastOrder.pagerCode }}</span></p>
            <p><b>Thời gian:</b> {{ new Date().toLocaleString('vi-VN') }}</p>
            <p><b>Thu ngân:</b> {{ currentUser ? currentUser.fullName : 'Admin' }}</p>
            <p><b>Loại đơn:</b> {{ lastOrder?.orderType }}</p>
            <div class="dashed-line"></div>
          </div>

          <table class="receipt-table">
            <thead>
              <tr>
                <th style="text-align: left;">Món</th>
                <th style="text-align: center;">SL</th>
                <th style="text-align: right;">Tiền</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="(item, idx) in lastOrder?.items" :key="idx">
                <td style="text-align: left;">
                  <b>{{ item.drinkName }}</b>
                  <div class="receipt-toppings" v-if="item.toppings.length">
                    <div v-for="t in item.toppings" :key="t.toppingId">+ {{ t.toppingName }}</div>
                  </div>
                </td>
                <td style="text-align: center; vertical-align: top;">{{ item.quantity }}</td>
                <td style="text-align: right; vertical-align: top;">{{ formatPrice(getItemTotal(item)) }}</td>
              </tr>
            </tbody>
          </table>

          <div class="dashed-line"></div>
          <div class="receipt-summary">
            <div class="row">
              <span>Tổng cộng:</span>
              <span>{{ formatPrice(lastOrder?.subTotal) }}</span>
            </div>
            <div class="row" v-if="lastOrder?.discount > 0">
              <span>Voucher giảm giá:</span>
              <span>- {{ formatPrice(lastOrder?.discount) }}</span>
            </div>
            <div class="row total-pay">
              <span>KHÁCH PHẢI TRẢ:</span>
              <span>{{ formatPrice(lastOrder?.total) }}</span>
            </div>
            <div class="row">
              <span>Thanh toán:</span>
              <span>{{ lastOrder?.paymentMethod }}</span>
            </div>
          </div>
          <div class="dashed-line"></div>
          <div class="receipt-footer">
            <p><b>Cảm ơn quý khách và hẹn gặp lại!</b></p>
            <p><i>Xin quý khách vui lòng giữ hóa đơn để kiểm tra đồ uống.</i></p>
          </div>
        </div>
      </div>
    </div>

  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue';
import api from '../../services/api';

const drinks = ref([]);
const categories = ref([]);
const toppings = ref([]);
const currentBill = ref([]);
const loading = ref(false);

const orderType = ref('Tại quán');
const paymentMethod = ref('Tiền mặt');
const pagerCode = ref(''); // MỚI: BIẾN LƯU THẺ RUNG
const selectedCat = ref(null);
const searchQuery = ref('');

const voucherCode = ref('');
const appliedVoucher = ref(null);

const currentUser = JSON.parse(localStorage.getItem('user'));

// Biến cho Popup
const showQRModal = ref(false);
const showReceiptModal = ref(false);
const lastOrder = ref(null); 

onMounted(async () => {
  const savedBill = localStorage.getItem('pos_bill');
  if (savedBill) {
    currentBill.value = JSON.parse(savedBill);
  }
  
  const savedSettings = localStorage.getItem('pos_settings');
  if (savedSettings) {
    const settings = JSON.parse(savedSettings);
    orderType.value = settings.orderType || 'Tại quán';
    paymentMethod.value = settings.paymentMethod || 'Tiền mặt';
    pagerCode.value = settings.pagerCode || ''; // MỚI: PHỤC HỒI BIẾN THẺ RUNG
  }

  try {
    const [resDrinks, resCats, resToppings] = await Promise.all([
      api.get('/drinks'),
      api.get('/categories'),
      api.get('/toppings/active') 
    ]);
    
    drinks.value = (resDrinks.data.data || []).filter(d => d.drinkActive);
    categories.value = resCats.data.data || [];
    toppings.value = resToppings.data.data || [];
  } catch (e) {
    console.error("Lỗi lấy dữ liệu POS", e);
  }
});

// ================= HÀM LƯU DỮ LIỆU TỰ ĐỘNG =================
const savePosBill = () => {
  localStorage.setItem('pos_bill', JSON.stringify(currentBill.value));
  localStorage.setItem('pos_settings', JSON.stringify({
    orderType: orderType.value,
    paymentMethod: paymentMethod.value,
    pagerCode: pagerCode.value // LƯU VÀO CACHE
  }));
};

const filteredDrinks = computed(() => {
  let result = drinks.value;
  if (selectedCat.value !== null) {
    result = result.filter(d => d.category?.categoryId === selectedCat.value);
  }
  if (searchQuery.value.trim() !== '') {
    const q = searchQuery.value.toLowerCase();
    result = result.filter(d => d.drinkName.toLowerCase().includes(q));
  }
  return result;
});

const addToBill = (drink) => {
  const existingItem = currentBill.value.find(
    item => item.drinkId === drink.drinkId && item.toppings.length === 0 && item.note === ''
  );

  if (existingItem) {
    existingItem.quantity += 1;
  } else {
    currentBill.value.push({ 
      ...drink, 
      quantity: 1,
      toppings: [], 
      note: '',
      selectedTopping: null 
    });
  }
  savePosBill(); 
};

const updateQty = (index, delta) => {
  if (currentBill.value[index].quantity + delta > 0) {
    currentBill.value[index].quantity += delta;
    savePosBill(); 
  }
};

const addTopping = (index) => {
  const item = currentBill.value[index];
  if (item.selectedTopping) {
    item.toppings.push({ ...item.selectedTopping });
    item.selectedTopping = null; 
    savePosBill(); 
  }
};

const removeTopping = (itemIndex, topIdx) => {
  currentBill.value[itemIndex].toppings.splice(topIdx, 1);
  savePosBill();
};

const removeItem = (index) => { 
  currentBill.value.splice(index, 1); 
  checkVoucherValid();
  savePosBill(); 
};

const clearBill = () => { 
  if(confirm("Xóa toàn bộ hóa đơn hiện tại?")) { 
    currentBill.value = []; 
    pagerCode.value = ''; // RESET THẺ RUNG
    removeVoucher(); 
    savePosBill();
  }
};

const getItemTotal = (item) => {
  let topSum = item.toppings.reduce((sum, t) => sum + (t.toppingPrice || 0), 0);
  return (item.drinkPrice + topSum) * item.quantity;
};

const subTotal = computed(() => {
  return currentBill.value.reduce((sum, item) => sum + getItemTotal(item), 0);
});

const finalTotal = computed(() => {
  let total = subTotal.value;
  if (appliedVoucher.value) total -= appliedVoucher.value.discountAmount;
  return total > 0 ? total : 0;
});

const applyVoucher = async () => {
  if (!voucherCode.value) return;
  try {
    const res = await api.post('/vouchers/check', { code: voucherCode.value.toUpperCase(), orderTotal: subTotal.value });
    if (res.data.status === 'success') { appliedVoucher.value = res.data.data; } 
    else { alert(res.data.message); }
  } catch (err) { alert("Mã không hợp lệ hoặc chưa đủ điều kiện!"); }
};

const removeVoucher = () => { appliedVoucher.value = null; voucherCode.value = ''; };

const checkVoucherValid = () => {
  if (appliedVoucher.value && subTotal.value < appliedVoucher.value.minOrderValue) {
    removeVoucher(); 
  }
};

const handleCheckoutClick = () => {
  if (paymentMethod.value === 'Chuyển khoản') {
    showQRModal.value = true; 
  } else {
    processCheckoutAPI(); 
  }
};

const processCheckoutAPI = async () => {
  loading.value = true;
  
  try {
    const billPayload = {
      orderType: orderType.value,
      paymentMethod: paymentMethod.value, 
      totalPrice: finalTotal.value,
      billStatus: 'Chờ pha chế', 
      pagerCode: pagerCode.value, // ĐẨY THẺ RUNG LÊN API
      user: currentUser ? { userId: currentUser.userId } : null, 
      voucher: appliedVoucher.value ? { voucherId: appliedVoucher.value.voucherId } : null
    };

    const resBill = await api.post('/bills', billPayload);
    const createdBill = resBill.data.data;

    for (const item of currentBill.value) {
      let baseUnitPrice = item.drinkPrice; 
      
      const detailPayload = {
        bill: { billId: createdBill.billId },
        drink: { drinkId: item.drinkId },
        bdQuantity: item.quantity, 
        unitPrice: baseUnitPrice,
        bdStatus: 'Chờ pha chế' 
      };
      
      const resDetail = await api.post('/bill-details', detailPayload);
      const createdDetail = resDetail.data.data;

      if (item.toppings && item.toppings.length > 0) {
        for (const top of item.toppings) {
          const toppingPayload = {
            billDetail: { bdId: createdDetail.bdId }, 
            topping: { toppingId: top.toppingId },    
            bdtPrice: top.toppingPrice                
          };
          try { await api.post('/bill-detail-toppings', toppingPayload); } catch(err) { }
        }
      }
    }

    lastOrder.value = {
      billCode: createdBill.billCode,
      orderType: orderType.value,
      pagerCode: pagerCode.value, // THÊM VÀO ĐỂ IN HÓA ĐƠN
      items: currentBill.value.map(item => ({ ...item })), 
      subTotal: subTotal.value,
      discount: appliedVoucher.value ? appliedVoucher.value.discountAmount : 0,
      total: finalTotal.value,
      paymentMethod: paymentMethod.value
    };

    currentBill.value = [];
    pagerCode.value = ''; // RESET LẠI Ô THẺ RUNG
    removeVoucher();
    localStorage.removeItem('pos_bill'); 

    showQRModal.value = false;
    showReceiptModal.value = true;

  } catch (err) {
    alert("Lỗi thanh toán: Vui lòng kiểm tra lại kết nối!");
  } finally {
    loading.value = false;
  }
};

const printReceipt = () => {
  window.print();
};

const closeReceiptAndReset = () => {
  showReceiptModal.value = false;
  lastOrder.value = null;
};

const formatPrice = (p) => new Intl.NumberFormat('vi-VN', { style: 'currency', currency: 'VND' }).format(p || 0);
</script>

<style scoped>
.pos-layout { display: flex; gap: 20px; height: calc(100vh - 20px); background: #eef2f5; padding: 10px 20px; overflow: hidden;}

.btn-return-admin { background: #34495e; color: white; border: none; padding: 8px 15px; border-radius: 6px; cursor: pointer; font-weight: bold; transition: 0.2s; font-size: 14px;}
.btn-return-admin:hover { background: #2c3e50; transform: translateX(-2px); }

.menu-col { flex: 6.5; display: flex; flex-direction: column; background: white; border-radius: 12px; padding: 20px; box-shadow: 0 4px 15px rgba(0,0,0,0.05);}
.header-search { display: flex; flex-direction: column; gap: 15px; margin-bottom: 20px; border-bottom: 2px solid #f4f6f8; padding-bottom: 15px;}
.header-search h2 { margin: 0; color: #2c3e50; font-size: 24px;}

.cat-filters { display: flex; gap: 10px; overflow-x: auto; padding-bottom: 5px;}
.cat-filters::-webkit-scrollbar { height: 4px; }
.cat-filters::-webkit-scrollbar-thumb { background: #bdc3c7; border-radius: 4px; }
.cat-btn { padding: 8px 15px; border-radius: 20px; border: 1px solid #bdc3c7; background: white; color: #7f8c8d; font-weight: bold; cursor: pointer; white-space: nowrap; transition: 0.2s;}
.cat-btn.active { background: #2980b9; color: white; border-color: #2980b9;}

.search-box-wrap { display: flex; align-items: center; background: #f4f6f8; border-radius: 8px; padding: 0 15px; border: 1px solid #dfe6e9;}
.search-box { flex: 1; padding: 12px 10px; border: none; background: transparent; font-size: 15px; outline: none;}

.drink-grid { flex: 1; display: grid; grid-template-columns: repeat(auto-fill, minmax(150px, 1fr)); gap: 15px; overflow-y: auto; padding-right: 5px; align-content: start;}
.drink-grid::-webkit-scrollbar { width: 6px; }
.drink-grid::-webkit-scrollbar-thumb { background: #bdc3c7; border-radius: 4px; }
.empty-menu { grid-column: 1 / -1; text-align: center; color: #95a5a6; padding: 50px; font-style: italic;}

.drink-card { background: white; border-radius: 10px; overflow: hidden; cursor: pointer; border: 1px solid #ecf0f1; transition: 0.15s; user-select: none;}
.drink-card:active { transform: scale(0.95); } 
.img-thumb { width: 100%; height: 110px; object-fit: cover; }
.info { padding: 10px; text-align: center; background: #fdfdfd;}
.info h4 { margin: 0 0 5px 0; font-size: 14px; color: #2c3e50; display: -webkit-box; -webkit-line-clamp: 2; -webkit-box-orient: vertical; overflow: hidden;}
.price { color: #d35400; font-weight: bold; margin: 0; font-size: 15px;}

.bill-col { flex: 3.5; background: white; border-radius: 12px; display: flex; flex-direction: column; box-shadow: 0 4px 20px rgba(0,0,0,0.08); overflow: hidden; }
.bill-title-bar { display: flex; justify-content: space-between; align-items: center; background: #2c3e50; padding: 15px 20px;}
.bill-title { margin: 0; color: white; font-size: 18px;}
.btn-clear { background: transparent; color: #e74c3c; border: 1px solid #e74c3c; padding: 5px 10px; border-radius: 5px; font-size: 12px; cursor: pointer; font-weight: bold;}
.btn-clear:hover { background: #e74c3c; color: white;}

.bill-items { flex: 1; overflow-y: auto; padding: 15px; background: #fafafa;}
.empty-msg { text-align: center; color: #bdc3c7; margin-top: 50px; font-size: 16px;}
.empty-icon { font-size: 40px; margin-bottom: 10px; opacity: 0.5;}

.bill-item { background: white; border: 1px solid #eee; border-radius: 8px; padding: 12px; margin-bottom: 12px; box-shadow: 0 2px 4px rgba(0,0,0,0.02);}
.item-main { display: flex; justify-content: space-between; align-items: center; margin-bottom: 10px; }
.item-name-qty { display: flex; align-items: center; gap: 10px; flex: 1;}
.item-name-qty b { color: #2c3e50; font-size: 15px; line-height: 1.2;}

.mini-qty { display: flex; align-items: center; background: #ecf0f1; border-radius: 20px; overflow: hidden; border: 1px solid #bdc3c7;}
.mini-qty button { background: transparent; border: none; width: 25px; height: 25px; font-weight: bold; color: #2980b9; cursor: pointer;}
.mini-qty button:disabled { color: #bdc3c7; cursor: not-allowed;}
.mini-qty span { font-size: 14px; font-weight: bold; width: 20px; text-align: center; color: #2c3e50;}

.item-price { font-weight: bold; color: #d35400; font-size: 15px;}

.item-actions { display: flex; gap: 8px; align-items: center;}
.topping-select { flex: 1; padding: 6px; border-radius: 4px; border: 1px solid #ddd; font-size: 12px; outline: none;}
.note-input { flex: 1; padding: 6px; border-radius: 4px; border: 1px solid #ddd; font-size: 12px; outline: none;}
.btn-remove { background: #fdf2e9; color: #e74c3c; border: 1px solid #fadbd8; border-radius: 4px; cursor: pointer; padding: 5px 8px; font-size: 12px;}

.topping-list { list-style: none; padding: 0; margin: 8px 0 0 0; font-size: 12px; color: #7f8c8d; border-top: 1px dashed #eee; padding-top: 8px;}
.topping-list li { display: flex; justify-content: space-between; margin-bottom: 3px; background: #fdfdfd; padding: 2px 5px; border-radius: 3px;}
.remove-topping { color: #e74c3c; cursor: pointer; font-weight: bold; font-size: 11px;}
.remove-topping:hover { text-decoration: underline;}

.checkout-zone { padding: 20px; background: white; border-top: 1px solid #eee; box-shadow: 0 -4px 10px rgba(0,0,0,0.03); z-index: 10;}

.pos-voucher { display: flex; gap: 10px; margin-bottom: 10px;}
.pos-voucher input { flex: 1; padding: 8px 10px; border: 1px solid #bdc3c7; border-radius: 5px; font-size: 13px; outline: none;}
.btn-sm-apply { background: #34495e; color: white; border: none; padding: 0 15px; border-radius: 5px; cursor: pointer; font-weight: bold; font-size: 12px;}
.btn-sm-remove { background: #e74c3c; color: white; border: none; padding: 0 15px; border-radius: 5px; cursor: pointer; font-weight: bold; font-size: 12px;}
.voucher-alert { color: #27ae60; font-size: 12px; font-weight: bold; margin-bottom: 10px; text-align: right;}

.summary-row { display: flex; justify-content: space-between; align-items: center; margin-bottom: 15px;}
.type-row span { font-size: 14px; color: #7f8c8d; font-weight: bold;}
.type-select { padding: 8px 12px; border-radius: 5px; border: 1px solid #bdc3c7; font-weight: bold; color: #2c3e50; outline: none; background: #f9f9f9;}

.total-row { border-top: 2px dashed #ecf0f1; padding-top: 15px;}
.total-row span { font-size: 16px; font-weight: 900; color: #2c3e50;}
.total-row b { color: #c0392b; font-size: 26px; }

.btn-pay { width: 100%; background: #27ae60; color: white; border: none; padding: 18px; font-weight: 900; font-size: 18px; border-radius: 8px; cursor: pointer; transition: 0.2s; box-shadow: 0 4px 10px rgba(39, 174, 96, 0.3); letter-spacing: 1px;}
.btn-pay:hover:not(:disabled) { background: #219653; transform: translateY(-2px);}
.btn-pay:disabled { background: #95a5a6; box-shadow: none; cursor: not-allowed;}

/* ================= MODAL & POPUP STYLES ================= */
.modal-overlay { position: fixed; top: 0; left: 0; width: 100vw; height: 100vh; background: rgba(0,0,0,0.6); display: flex; justify-content: center; align-items: center; z-index: 2000; }
.qr-modal-content { background: white; padding: 30px; border-radius: 12px; width: 350px; box-shadow: 0 10px 30px rgba(0,0,0,0.3); }
.qr-box { background: #f4f6f8; padding: 15px; border-radius: 8px; display: flex; justify-content: center; margin-bottom: 15px;}
.qr-image { width: 250px; height: 250px; mix-blend-mode: multiply; }
.modal-actions { display: flex; gap: 10px; margin-top: 20px; }
.btn-cancel { flex: 1; background: #ecf0f1; color: #7f8c8d; border: none; padding: 12px; border-radius: 6px; font-weight: bold; cursor: pointer; transition: 0.2s;}
.btn-cancel:hover { background: #bdc3c7; color: white; }

/* ================= HÓA ĐƠN GIẤY (THERMAL RECEIPT) ================= */
.receipt-preview-wrap { display: flex; flex-direction: column; align-items: center; gap: 15px; }
.receipt-actions { display: flex; gap: 10px; width: 100%; justify-content: center;}
.btn-print { background: #2980b9; color: white; padding: 10px 20px; border: none; border-radius: 5px; font-weight: bold; cursor: pointer;}
.btn-cancel-print { background: #e74c3c; color: white; padding: 10px 20px; border: none; border-radius: 5px; font-weight: bold; cursor: pointer;}

.receipt-paper { background: white; width: 320px; padding: 20px; font-family: 'Courier New', Courier, monospace; font-size: 13px; color: #000; box-shadow: 0 5px 15px rgba(0,0,0,0.2); }
.receipt-header { text-align: center; margin-bottom: 15px; }
.receipt-header h2 { margin: 0 0 5px 0; font-size: 22px; font-weight: bold;}
.receipt-header p { margin: 2px 0; font-size: 12px; }
.receipt-header h3 { margin: 15px 0 5px 0; font-size: 16px; }
.dashed-line { border-top: 1px dashed #000; margin: 10px 0; }

.receipt-table { width: 100%; border-collapse: collapse; margin-bottom: 10px; }
.receipt-table th { border-bottom: 1px solid #000; padding-bottom: 5px; margin-bottom: 5px;}
.receipt-table td { padding: 5px 0; }
.receipt-toppings { font-size: 11px; padding-left: 10px; font-style: italic; }

.receipt-summary .row { display: flex; justify-content: space-between; margin-bottom: 5px; }
.total-pay { font-weight: bold; font-size: 16px; margin: 10px 0 !important; }
.receipt-footer { text-align: center; margin-top: 15px; font-size: 12px; }

/* CSS IN ẤN ĐẶC BIỆT: Chỉ in tờ bill, giấu hết website đi */
@media print {
  body * { visibility: hidden; }
  #print-area, #print-area * { visibility: visible; }
  #print-area { position: absolute; left: 0; top: 0; width: 100%; padding: 0; margin: 0; box-shadow: none;}
  .no-print { display: none !important; }
}
</style>