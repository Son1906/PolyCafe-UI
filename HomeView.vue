<template>
  <div class="customer-home">
    <div class="banner">
      <h1>PolyCafe Chào Bạn!</h1>
      <p>Thưởng thức hương vị tuyệt hảo ngay hôm nay.</p>
    </div>

    <div class="filter-bar" v-if="categories.length > 0">
      <button 
        class="cat-btn" 
        :class="{ active: selectedCategory === null }" 
        @click="filterMenu(null)"
      >Tất cả</button>
      <button 
        class="cat-btn" 
        v-for="cat in categories" 
        :key="cat.categoryId"
        :class="{ active: selectedCategory === cat.categoryId }" 
        @click="filterMenu(cat.categoryId)"
      >
        {{ cat.categoryName }}
      </button>
    </div>

    <h2 class="section-title">☕ Thực Đơn Của Quán</h2>
    
    <div class="grid">
      <div v-if="filteredDrinks.length === 0" class="empty-state">
        <p>Hiện chưa có thức uống nào trong danh mục này.</p>
      </div>
      
      <div class="card" v-for="d in filteredDrinks" :key="d.drinkId">
        <img :src="d.image || 'https://images.unsplash.com/photo-1559525839-b184a4d698c7?w=500&q=80'" class="img" alt="Drink" />
        <div class="card-body">
          <h3>{{ d.drinkName }}</h3>
          <p class="desc">{{ d.description || 'Thức uống ngon tuyệt đỉnh, đậm đà hương vị truyền thống.' }}</p>
          <div class="price-row">
            <span class="price">{{ formatPrice(d.drinkPrice) }}</span>
            <button class="btn-add" @click="openOrderModal(d)">CHỌN MUA</button>
          </div>
        </div>
      </div>
    </div>

    <div class="modal-overlay" v-if="showModal">
      <div class="modal-content">
        <button class="btn-close" @click="closeModal">❌</button>
        
        <div class="modal-header">
          <img :src="selectedDrink.image || 'https://images.unsplash.com/photo-1559525839-b184a4d698c7?w=500&q=80'" class="modal-img" />
          <div class="modal-title">
            <h3>{{ selectedDrink.drinkName }}</h3>
            <p class="avg-rating" v-if="avgRating > 0">⭐ {{ avgRating }} / 5.0</p>
            <p class="modal-price">{{ formatPrice(selectedDrink.drinkPrice) }}</p>
          </div>
        </div>

        <div class="modal-tabs">
          <button class="tab-btn" :class="{active: activeTab === 'order'}" @click="activeTab = 'order'">🛒 Chọn Món</button>
          <button class="tab-btn" :class="{active: activeTab === 'review'}" @click="activeTab = 'review'">⭐ Đánh giá ({{ reviews.length }})</button>
        </div>

        <div class="modal-body" v-if="activeTab === 'order'">
          <div class="options-section" v-if="availableToppings.length > 0">
            <h4>Thêm Topping (Tùy chọn)</h4>
            <div class="topping-list">
              <label class="topping-item" v-for="t in availableToppings" :key="t.toppingId">
                <input type="checkbox" :value="t" v-model="orderOptions.toppings" />
                <span class="t-name">{{ t.toppingName }}</span>
                <span class="t-price">+{{ formatPrice(t.toppingPrice || t.price) }}</span>
              </label>
            </div>
          </div>

          <div class="options-section">
            <div class="form-group">
              <label>Ghi chú cho quán (Ít đá, nhiều ngọt...)</label>
              <textarea v-model="orderOptions.note" rows="2" placeholder="Nhập ghi chú của bạn..."></textarea>
            </div>
            
            <div class="qty-wrap">
              <label>Số lượng:</label>
              <div class="qty-control">
                <button @click="updateModalQty(-1)" :disabled="orderOptions.quantity <= 1">-</button>
                <input type="number" readonly :value="orderOptions.quantity" />
                <button @click="updateModalQty(1)">+</button>
              </div>
            </div>
          </div>
          
          <div class="modal-footer">
            <div class="total-preview">
              Tạm tính: <b>{{ formatPrice(calculateCurrentTotal()) }}</b>
            </div>
            <button class="btn-add-cart" @click="addToCart">
              🛒 THÊM VÀO GIỎ
            </button>
          </div>
        </div>

        <div class="modal-body review-tab-content" v-else>
          <div class="review-form" v-if="currentUser">
            <h4>👉 Viết đánh giá của bạn</h4>
            <div class="star-rating">
              <span 
                v-for="star in 5" 
                :key="star" 
                @click="newReview.rating = star"
                class="star-btn"
                :class="{ active: star <= newReview.rating }"
              >
                {{ star <= newReview.rating ? '⭐' : '☆' }}
              </span>
            </div>
            <textarea v-model="newReview.content" rows="3" placeholder="Ly nước này thế nào? Hãy chia sẻ nhé..."></textarea>
            <button class="btn-submit-review" @click="submitReview" :disabled="loadingReview">
              {{ loadingReview ? 'Đang gửi...' : 'Gửi Đánh Giá' }}
            </button>
          </div>

          <div v-else class="not-logged-in">
            <p>Vui lòng <b>Đăng nhập</b> để có thể đánh giá món nước này.</p>
            <button class="btn-secondary" @click="$router.push('/login')">Đi tới Đăng nhập</button>
          </div>

          <div class="review-list">
            <h4>Khách hàng nói gì?</h4>
            <div v-if="reviews.length === 0" class="empty-reviews">
              Món này chưa có đánh giá nào. Hãy là người đầu tiên review nhé!
            </div>
            <div class="review-item" v-for="r in reviews" :key="r.reviewId">
              <div class="r-header">
                <b>{{ r.user?.fullName || 'Khách' }}</b>
                <span class="r-date">{{ new Date(r.createdAt).toLocaleDateString('vi-VN') }}</span>
              </div>
              <div class="r-stars"><span v-for="s in r.rating" :key="s">⭐</span></div>
              <p class="r-content">{{ r.content }}</p>
            </div>
          </div>
        </div>
      </div>
    </div>

    <div class="toast-msg" :class="{ show: toast.show }">
      ✅ {{ toast.message }}
    </div>

  </div>
</template>

<script setup>
import { ref, reactive, computed, onMounted } from 'vue';
import api from '../../services/api';

const drinks = ref([]);
const categories = ref([]);
const availableToppings = ref([]);
const selectedCategory = ref(null);

// State Modal Chốt đơn
const showModal = ref(false);
const activeTab = ref('order');
const selectedDrink = ref(null);
const orderOptions = ref({ quantity: 1, toppings: [], note: '' });

// State Toast Thông Báo
const toast = reactive({ show: false, message: '' });

const showToast = (msg) => {
  toast.message = msg;
  toast.show = true;
  setTimeout(() => { toast.show = false; }, 3000);
};

// State Đánh Giá (Review)
const reviews = ref([]);
const avgRating = ref(0);
const currentUser = JSON.parse(localStorage.getItem('user'));
const newReview = reactive({ rating: 5, content: '' });
const loadingReview = ref(false);

onMounted(async () => {
  try {
    const resDrinks = await api.get('/drinks');
    drinks.value = (resDrinks.data.data || []).filter(d => d.drinkActive);

    const resCats = await api.get('/categories');
    categories.value = resCats.data.data || [];

    const resTops = await api.get('/toppings/active');
    availableToppings.value = resTops.data.data || [];
  } catch (err) {
    console.error("Lỗi tải dữ liệu Menu:", err);
  }
});

const filteredDrinks = computed(() => {
  if (selectedCategory.value === null) return drinks.value;
  return drinks.value.filter(d => d.category?.categoryId === selectedCategory.value);
});

const filterMenu = (catId) => { selectedCategory.value = catId; };

const openOrderModal = async (drink) => {
  selectedDrink.value = drink;
  orderOptions.value = { quantity: 1, toppings: [], note: '' };
  activeTab.value = 'order';
  showModal.value = true;

  try {
    const resRev = await api.get(`/reviews/drink/${drink.drinkId}`);
    reviews.value = resRev.data.data || [];
    
    const resAvg = await api.get(`/reviews/drink/${drink.drinkId}/average`);
    avgRating.value = resAvg.data.data || 0;
  } catch (err) { console.error("Lỗi lấy review:", err); }
};

const closeModal = () => { showModal.value = false; };

const updateModalQty = (delta) => {
  if (orderOptions.value.quantity + delta > 0) {
    orderOptions.value.quantity += delta;
  }
};

const calculateCurrentTotal = () => {
  if (!selectedDrink.value) return 0;
  let basePrice = selectedDrink.value.drinkPrice;
  let toppingsPrice = orderOptions.value.toppings.reduce((sum, t) => sum + (t.toppingPrice || t.price || 0), 0);
  return (basePrice + toppingsPrice) * orderOptions.value.quantity;
};

// Logic Submit Review
const submitReview = async () => {
  if (!newReview.content.trim()) return alert("Vui lòng nhập nội dung đánh giá!");
  loadingReview.value = true;
  try {
    await api.post('/reviews', {
      rating: newReview.rating,
      content: newReview.content,
      drink: { drinkId: selectedDrink.value.drinkId },
      user: { userId: currentUser.userId }
    });
    
    const resRev = await api.get(`/reviews/drink/${selectedDrink.value.drinkId}`);
    reviews.value = resRev.data.data || [];
    const resAvg = await api.get(`/reviews/drink/${selectedDrink.value.drinkId}/average`);
    avgRating.value = resAvg.data.data || 0;

    newReview.content = '';
    newReview.rating = 5;
    alert('Cảm ơn bạn đã đánh giá!');
  } catch (err) { 
    alert('Lỗi gửi đánh giá!'); 
  } finally {
    loadingReview.value = false;
  }
};

// ================= THUẬT TOÁN THÊM GIỎ HÀNG THÔNG MINH =================
const addToCart = () => {
  let cart = JSON.parse(localStorage.getItem('cart')) || [];
  
  // Nén danh sách Topping thành chuỗi để so sánh xem khách có order giống hệt ly trước không
  const currentToppingIds = orderOptions.value.toppings.map(t => t.toppingId).sort().join(',');

  // Tìm trong giỏ xem đã có ly nào y chang chưa (cùng món, cùng topping, cùng ghi chú)
  const existingItemIndex = cart.findIndex(item => {
    const itemToppingIds = (item.toppings || []).map(t => t.toppingId).sort().join(',');
    return item.drinkId === selectedDrink.value.drinkId &&
           item.note === orderOptions.value.note &&
           itemToppingIds === currentToppingIds;
  });

  if (existingItemIndex > -1) {
    // Nếu y hệt -> Chỉ tăng số lượng
    cart[existingItemIndex].quantity += orderOptions.value.quantity;
  } else {
    // Nếu khác -> Tạo dòng mới trong giỏ
    const cartItem = {
      drinkId: selectedDrink.value.drinkId,
      drinkName: selectedDrink.value.drinkName,
      drinkPrice: selectedDrink.value.drinkPrice,
      image: selectedDrink.value.image,
      quantity: orderOptions.value.quantity,
      toppings: [...orderOptions.value.toppings],
      note: orderOptions.value.note
    };
    cart.push(cartItem);
  }

  localStorage.setItem('cart', JSON.stringify(cart));
  
  // Bắn tín hiệu để Header (CustomerLayout) tự động cập nhật số lượng giỏ hàng
  window.dispatchEvent(new Event('cart-updated'));
  
  closeModal();
  
  // Bật Toast thông báo cho khách vui
  showToast(`Đã thêm ${orderOptions.value.quantity} ly ${selectedDrink.value.drinkName} vào giỏ!`);
};

const formatPrice = (p) => new Intl.NumberFormat('vi-VN', { style: 'currency', currency: 'VND' }).format(p || 0);
</script>

<style scoped>
.customer-home { max-width: 1200px; margin: 0 auto; padding-bottom: 50px;}

.banner { background: linear-gradient(135deg, #8B4513, #D2691E); color: white; padding: 60px 20px; text-align: center; border-radius: 15px; margin-bottom: 30px; box-shadow: 0 10px 20px rgba(139, 69, 19, 0.2);}
.banner h1 { margin: 0 0 10px 0; font-size: 40px; text-shadow: 2px 2px 4px rgba(0,0,0,0.3); letter-spacing: 1px;}
.banner p { font-size: 18px; opacity: 0.9; margin: 0;}

/* Bộ lọc danh mục */
.filter-bar { display: flex; gap: 15px; margin-bottom: 35px; overflow-x: auto; padding-bottom: 10px; }
.filter-bar::-webkit-scrollbar { height: 6px; }
.filter-bar::-webkit-scrollbar-thumb { background: #bdc3c7; border-radius: 3px; }
.cat-btn { background: white; border: 1px solid #e0e0e0; color: #7f8c8d; padding: 10px 20px; border-radius: 30px; font-weight: bold; cursor: pointer; white-space: nowrap; transition: 0.3s; box-shadow: 0 2px 5px rgba(0,0,0,0.05);}
.cat-btn:hover { border-color: #f39c12; color: #f39c12; }
.cat-btn.active { background: #f39c12; color: white; border-color: #f39c12; box-shadow: 0 4px 10px rgba(243, 156, 18, 0.3);}

.section-title { color: #2c3e50; border-left: 5px solid #f39c12; padding-left: 15px; margin-bottom: 25px; font-size: 26px;}

.empty-state { grid-column: 1 / -1; text-align: center; padding: 50px; background: white; border-radius: 12px; color: #7f8c8d; font-style: italic; font-size: 16px;}

/* Lưới Menu */
.grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(260px, 1fr)); gap: 30px; }
.card { background: white; border-radius: 12px; overflow: hidden; box-shadow: 0 5px 15px rgba(0,0,0,0.05); transition: 0.3s; position: relative; border: 1px solid #f9f9f9;}
.card:hover { transform: translateY(-8px); box-shadow: 0 15px 30px rgba(0,0,0,0.1); border-color: #fcecd4;}
.img { width: 100%; height: 200px; object-fit: cover; transition: 0.5s;}
.card:hover .img { transform: scale(1.05); }

.card-body { padding: 20px; background: white; position: relative; z-index: 2;}
.card-body h3 { margin: 0 0 10px 0; font-size: 19px; color: #2c3e50; font-weight: 700;}
.desc { color: #7f8c8d; font-size: 13px; margin-bottom: 20px; line-height: 1.5; display: -webkit-box; -webkit-line-clamp: 2; -webkit-box-orient: vertical; overflow: hidden; height: 39px;}
.price-row { display: flex; justify-content: space-between; align-items: center; border-top: 1px solid #f0f0f0; padding-top: 15px;}
.price { color: #d35400; font-weight: 800; font-size: 20px; }

.btn-add { background: #f39c12; color: white; border: none; padding: 10px 18px; border-radius: 8px; font-weight: bold; font-size: 14px; cursor: pointer; transition: 0.3s; box-shadow: 0 2px 5px rgba(243, 156, 18, 0.3);}
.btn-add:hover { background: #e67e22; transform: translateY(-2px); box-shadow: 0 4px 8px rgba(243, 156, 18, 0.4);}

/* ================= MODAL CHỐT ĐƠN ================= */
.modal-overlay { position: fixed; top: 0; left: 0; width: 100vw; height: 100vh; background: rgba(0,0,0,0.6); display: flex; justify-content: center; align-items: flex-end; z-index: 1000; padding-bottom: 2vh;}
.modal-content { background: white; width: 500px; max-width: 100%; border-radius: 20px; overflow: hidden; position: relative; display: flex; flex-direction: column; max-height: 90vh; animation: slideUp 0.3s ease;}

@keyframes slideUp { from { transform: translateY(100%); } to { transform: translateY(0); } }

.btn-close { position: absolute; top: 15px; right: 15px; background: rgba(255,255,255,0.8); border: none; font-size: 18px; width: 35px; height: 35px; border-radius: 50%; cursor: pointer; z-index: 10; display: flex; align-items: center; justify-content: center;}
.btn-close:hover { background: #eee;}

.modal-header { display: flex; gap: 20px; padding: 25px; background: #fdfaf4; border-bottom: 1px solid #eee; align-items: center;}
.modal-img { width: 100px; height: 100px; border-radius: 12px; object-fit: cover; box-shadow: 0 4px 10px rgba(0,0,0,0.1);}
.modal-title h3 { margin: 0 0 8px 0; font-size: 22px; color: #2c3e50; }
.avg-rating { margin: 0 0 5px 0; color: #f39c12; font-weight: bold; font-size: 14px;}
.modal-price { margin: 0; color: #d35400; font-size: 20px; font-weight: bold; }

/* Tabs */
.modal-tabs { display: flex; background: white; border-bottom: 1px solid #eee; position: sticky; top: 0; z-index: 5;}
.tab-btn { flex: 1; padding: 15px; background: none; border: none; border-bottom: 3px solid transparent; cursor: pointer; font-weight: bold; color: #7f8c8d;}
.tab-btn.active { color: #f39c12; border-bottom-color: #f39c12; }

/* Modal Body Chung */
.modal-body { padding: 25px; overflow-y: auto; display: flex; flex-direction: column;}
.options-section { margin-bottom: 25px; }
.options-section h4 { margin: 0 0 15px 0; color: #34495e; font-size: 16px; border-bottom: 2px solid #eee; padding-bottom: 8px;}

/* Chọn Topping */
.topping-list { display: flex; flex-direction: column; gap: 12px; }
.topping-item { display: flex; align-items: center; cursor: pointer; background: #f9f9f9; padding: 12px 15px; border-radius: 8px; border: 1px solid #eee; transition: 0.2s;}
.topping-item:hover { border-color: #f39c12; background: #fffcf8;}
.topping-item input { margin-right: 15px; transform: scale(1.2); accent-color: #f39c12;}
.t-name { flex: 1; font-weight: 600; color: #2c3e50;}
.t-price { color: #e67e22; font-weight: bold; font-size: 14px;}

/* Ghi chú & Số lượng */
.form-group label, .qty-wrap label { display: block; font-weight: bold; color: #34495e; margin-bottom: 8px;}
.form-group textarea { width: 100%; padding: 12px; border: 1px solid #ddd; border-radius: 8px; box-sizing: border-box; font-family: inherit; resize: vertical;}
.form-group textarea:focus { outline: none; border-color: #f39c12; }

.qty-wrap { display: flex; justify-content: space-between; align-items: center; margin-top: 20px; background: #f9f9f9; padding: 15px; border-radius: 8px;}
.qty-control { display: flex; align-items: center; border: 1px solid #ccc; border-radius: 8px; overflow: hidden; background: white;}
.qty-control button { background: white; border: none; width: 40px; height: 40px; cursor: pointer; font-weight: bold; font-size: 20px; color: #f39c12; transition: 0.2s;}
.qty-control button:hover:not(:disabled) { background: #fef5e6; }
.qty-control button:disabled { opacity: 0.3; cursor: not-allowed; }
.qty-control input { width: 50px; text-align: center; border: none; border-left: 1px solid #eee; border-right: 1px solid #eee; font-weight: bold; font-size: 18px; pointer-events: none;}

.modal-footer { margin-top: 15px; padding-top: 15px; border-top: 1px solid #eee;}
.total-preview { text-align: right; color: #7f8c8d; font-size: 15px; margin-bottom: 15px;}
.total-preview b { color: #d35400; font-size: 22px;}
.btn-add-cart { width: 100%; background: #27ae60; color: white; border: none; padding: 18px; border-radius: 10px; font-weight: bold; font-size: 16px; cursor: pointer; transition: 0.3s; box-shadow: 0 4px 10px rgba(39, 174, 96, 0.3);}
.btn-add-cart:hover { background: #2ecc71; transform: translateY(-2px);}

/* TAB ĐÁNH GIÁ */
.review-tab-content { background: #fafafa; }
.review-form { background: white; padding: 20px; border-radius: 10px; margin-bottom: 25px; box-shadow: 0 2px 8px rgba(0,0,0,0.05);}
.review-form h4 { margin: 0 0 15px 0; color: #2c3e50;}
.star-rating { margin-bottom: 10px; }
.star-btn { font-size: 28px; cursor: pointer; color: #ddd; transition: 0.2s;}
.star-btn.active { color: #f1c40f;}
.btn-submit-review { background: #3498db; color: white; border: none; padding: 10px 20px; border-radius: 8px; font-weight: bold; cursor: pointer; margin-top: 10px;}
.btn-submit-review:hover { background: #2980b9;}

.not-logged-in { text-align: center; padding: 20px; background: #fef2f2; color: #c0392b; border-radius: 10px; border: 1px dashed #e74c3c; margin-bottom: 25px;}
.btn-secondary { background: #34495e; color: white; border: none; padding: 8px 15px; border-radius: 5px; cursor: pointer; margin-top: 10px;}

.review-list h4 { color: #34495e; border-bottom: 2px solid #eee; padding-bottom: 10px; margin-bottom: 15px;}
.empty-reviews { text-align: center; color: #95a5a6; font-style: italic; padding: 20px 0;}
.review-item { padding: 15px; background: white; border-radius: 8px; margin-bottom: 15px; box-shadow: 0 2px 5px rgba(0,0,0,0.02);}
.r-header { display: flex; justify-content: space-between; margin-bottom: 5px;}
.r-date { font-size: 12px; color: #95a5a6;}
.r-stars { font-size: 12px; margin-bottom: 8px;}
.r-content { margin: 0; font-size: 14px; color: #2c3e50; line-height: 1.5;}

/* ================= TOAST THÔNG BÁO ================= */
.toast-msg {
  position: fixed;
  bottom: -100px; /* Ẩn đi */
  left: 50%;
  transform: translateX(-50%);
  background: #2c3e50;
  color: white;
  padding: 12px 25px;
  border-radius: 30px;
  font-weight: bold;
  font-size: 15px;
  box-shadow: 0 10px 30px rgba(0,0,0,0.3);
  transition: all 0.4s cubic-bezier(0.68, -0.55, 0.265, 1.55);
  z-index: 9999;
}

.toast-msg.show {
  bottom: 30px; /* Trượt lên */
}
</style>