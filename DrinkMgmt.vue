<template>
  <div class="mgmt-page">
    <div class="header">
      <h2>🥤 Quản Lý Đồ Uống & Công Thức</h2>
      <button class="btn-primary" @click="openAddModal">+ Thêm Đồ Uống Mới</button>
    </div>

    <table class="table">
      <thead>
        <tr>
          <th>ID</th>
          <th>Hình ảnh</th>
          <th>Tên Đồ Uống</th>
          <th>Danh Mục</th>
          <th>Giá Bán</th>
          <th>Trạng Thái</th>
          <th>Thao tác</th>
        </tr>
      </thead>
      <tbody>
        <tr v-if="drinks.length === 0">
          <td colspan="7" class="text-center">Chưa có đồ uống nào.</td>
        </tr>
        <tr v-for="d in drinks" :key="d.drinkId">
          <td class="code">#{{ d.drinkId }}</td>
          <td>
            <img :src="d.image || 'https://via.placeholder.com/50?text=Drink'" class="drink-thumb" />
          </td>
          <td><b>{{ d.drinkName }}</b></td>
          <td>{{ d.category?.categoryName || 'Không có' }}</td>
          <td class="price">{{ formatPrice(d.drinkPrice) }}</td>
          <td>
            <span :class="d.drinkActive ? 'badge-active' : 'badge-inactive'">
              {{ d.drinkActive ? 'Đang bán' : 'Ngừng bán' }}
            </span>
          </td>
          <td>
            <!-- NÚT CÀI ĐẶT CÔNG THỨC -->
            <button class="btn-recipe" @click="openRecipeModal(d)">🧪 Công thức</button>
            <button class="btn-edit" @click="openEditModal(d)">✏️ Sửa</button>
            <button class="btn-delete" @click="deleteDrink(d.drinkId)">🗑️ Xóa</button>
          </td>
        </tr>
      </tbody>
    </table>

    <!-- MODAL THÊM / SỬA ĐỒ UỐNG (CƠ BẢN) -->
    <div class="modal-overlay" v-if="showModal">
      <div class="modal-content">
        <h3>{{ isEditing ? 'Cập Nhật Đồ Uống' : 'Thêm Đồ Uống Mới' }}</h3>
        <form @submit.prevent="handleSubmit" class="form-grid">
          <div class="form-group full-width">
            <label>Tên đồ uống <span class="required">*</span></label>
            <input type="text" v-model="formData.drinkName" required />
          </div>
          <div class="form-row">
            <div class="form-group flex-1">
              <label>Danh mục <span class="required">*</span></label>
              <select v-model="formData.category.categoryId" required>
                <option value="" disabled>-- Chọn danh mục --</option>
                <option v-for="c in categories" :key="c.categoryId" :value="c.categoryId">{{ c.categoryName }}</option>
              </select>
            </div>
            <div class="form-group flex-1">
              <label>Giá bán (VNĐ) <span class="required">*</span></label>
              <input type="number" min="0" v-model="formData.drinkPrice" required />
            </div>
          </div>
          <div class="form-group full-width">
            <label>Link ảnh (URL)</label>
            <input type="text" v-model="formData.image" placeholder="https://..." />
          </div>
          <div class="form-group full-width" v-if="isEditing">
            <label>
              <input type="checkbox" v-model="formData.drinkActive" /> Cho phép bán (Active)
            </label>
          </div>
          <div class="modal-actions full-width">
            <button type="button" class="btn-cancel" @click="showModal = false">Hủy</button>
            <button type="submit" class="btn-save" :disabled="loading">{{ loading ? 'Đang lưu...' : '💾 Lưu Lại' }}</button>
          </div>
        </form>
      </div>
    </div>

    <!-- MODAL CÀI ĐẶT CÔNG THỨC PHA CHẾ (RECIPE) -->
    <div class="modal-overlay" v-if="showRecipeModal">
      <div class="modal-content recipe-modal">
        <div class="modal-header">
          <h3>🧪 Công Thức: <span class="highlight">{{ selectedDrink?.drinkName }}</span></h3>
          <button class="btn-close" @click="showRecipeModal = false">❌</button>
        </div>

        <div class="recipe-container">
          <!-- Danh sách nguyên liệu trong công thức -->
          <table class="table recipe-table">
            <thead>
              <tr>
                <th>Nguyên liệu (Từ kho)</th>
                <th>Định mức tiêu hao / 1 Ly</th>
                <th>Xóa</th>
              </tr>
            </thead>
            <tbody>
              <tr v-if="currentRecipes.length === 0">
                <td colspan="3" class="text-center">Chưa thiết lập định lượng cho món này.</td>
              </tr>
              <tr v-for="r in currentRecipes" :key="r.recipeId">
                <td><b>{{ r.ingredient?.iName }}</b></td>
                <td class="text-green"><b>{{ r.quantityRequired }}</b> {{ r.ingredient?.unit }}</td>
                <td>
                  <button class="btn-delete-sm" @click="removeRecipe(r.recipeId)">❌</button>
                </td>
              </tr>
            </tbody>
          </table>

          <!-- Form thêm nguyên liệu vào công thức -->
          <div class="add-recipe-box">
            <h4>Thêm định lượng nguyên liệu</h4>
            <form @submit.prevent="addRecipe" class="recipe-form">
              <div class="form-group flex-2">
                <select v-model="recipeForm.ingredientId" @change="updateUnitHint" required>
                  <option value="" disabled>-- Chọn nguyên liệu --</option>
                  <option v-for="ing in ingredients" :key="ing.iId" :value="ing.iId">
                    {{ ing.iName }}
                  </option>
                </select>
              </div>
              <div class="form-group flex-1 relative">
                <input type="number" step="0.1" min="0.1" v-model="recipeForm.quantityRequired" placeholder="Số lượng..." required />
                <span class="unit-hint">{{ unitHint }}</span>
              </div>
              <button type="submit" class="btn-primary" :disabled="loadingRecipe">Thêm</button>
            </form>
          </div>
          
        </div>
      </div>
    </div>

  </div>
</template>

<script setup>
import { ref, reactive, onMounted } from 'vue';
import api from '../../services/api';

const drinks = ref([]);
const categories = ref([]);
const ingredients = ref([]);

// State quản lý Đồ uống
const showModal = ref(false);
const isEditing = ref(false);
const currentDrinkId = ref(null);
const loading = ref(false);
const formData = reactive({
  drinkName: '',
  drinkPrice: 0,
  image: '',
  drinkActive: true,
  category: { categoryId: '' }
});

// State quản lý Công thức (Recipe)
const showRecipeModal = ref(false);
const selectedDrink = ref(null);
const currentRecipes = ref([]);
const loadingRecipe = ref(false);
const unitHint = ref('');

const recipeForm = reactive({
  ingredientId: '',
  quantityRequired: ''
});

onMounted(async () => {
  fetchDrinks();
  try {
    const [resCat, resIng] = await Promise.all([
      api.get('/categories'),
      api.get('/ingredients')
    ]);
    categories.value = resCat.data.data || [];
    ingredients.value = resIng.data.data || [];
  } catch (e) {
    console.error("Lỗi lấy danh mục/nguyên liệu", e);
  }
});

const fetchDrinks = async () => {
  try {
    const res = await api.get('/drinks');
    drinks.value = res.data.data || [];
  } catch (err) {
    console.error("Lỗi lấy đồ uống", err);
  }
};

// ================= CRUD ĐỒ UỐNG =================
const openAddModal = () => {
  isEditing.value = false;
  Object.assign(formData, { drinkName: '', drinkPrice: 0, image: '', drinkActive: true, category: { categoryId: '' } });
  showModal.value = true;
};

const openEditModal = (d) => {
  isEditing.value = true;
  currentDrinkId.value = d.drinkId;
  Object.assign(formData, { 
    drinkName: d.drinkName, 
    drinkPrice: d.drinkPrice, 
    image: d.image, 
    drinkActive: d.drinkActive, 
    category: { categoryId: d.category?.categoryId || '' } 
  });
  showModal.value = true;
};

const handleSubmit = async () => {
  loading.value = true;
  try {
    if (isEditing.value) {
      await api.put(`/drinks/${currentDrinkId.value}`, formData);
    } else {
      await api.post('/drinks', formData);
    }
    showModal.value = false;
    fetchDrinks();
  } catch (err) {
    alert("Lỗi lưu đồ uống!");
  } finally {
    loading.value = false;
  }
};

const deleteDrink = async (id) => {
  if (confirm("Chắc chắn xóa món này? Tất cả công thức liên quan cũng sẽ bị xóa!")) {
    try {
      // Gọi BE xóa Drink (BE nên tự cascade xóa các dòng Recipe tương ứng)
      await api.delete(`/drinks/${id}`);
      fetchDrinks();
    } catch (err) {
      alert("Lỗi xóa! Có thể do món này đã có trong Hóa đơn.");
    }
  }
};

// ================= RECIPE (CÔNG THỨC) =================
const openRecipeModal = async (drink) => {
  selectedDrink.value = drink;
  recipeForm.ingredientId = '';
  recipeForm.quantityRequired = '';
  unitHint.value = '';
  showRecipeModal.value = true;
  await fetchRecipesForDrink(drink.drinkId);
};

const fetchRecipesForDrink = async (drinkId) => {
  try {
    // Gọi API lấy list công thức của 1 món
    const res = await api.get(`/recipes/drink/${drinkId}`);
    currentRecipes.value = res.data.data || [];
  } catch (err) {
    console.error("Lỗi tải công thức", err);
    currentRecipes.value = [];
  }
};

const updateUnitHint = () => {
  const ing = ingredients.value.find(i => i.iId === recipeForm.ingredientId);
  unitHint.value = ing ? ing.unit : '';
};

const addRecipe = async () => {
  if (!recipeForm.ingredientId || !recipeForm.quantityRequired) return;
  loadingRecipe.value = true;
  
  try {
    const payload = {
      drink: { drinkId: selectedDrink.value.drinkId },
      ingredient: { iId: recipeForm.ingredientId }, // Chú ý iId
      quantityRequired: recipeForm.quantityRequired
    };
    
    // API lưu công thức
    await api.post('/recipes', payload);
    
    // Reset form
    recipeForm.ingredientId = '';
    recipeForm.quantityRequired = '';
    unitHint.value = '';
    
    // Tải lại bảng công thức
    await fetchRecipesForDrink(selectedDrink.value.drinkId);
  } catch (err) {
    alert("Lỗi thêm công thức!");
  } finally {
    loadingRecipe.value = false;
  }
};

const removeRecipe = async (recipeId) => {
  try {
    await api.delete(`/recipes/${recipeId}`);
    await fetchRecipesForDrink(selectedDrink.value.drinkId);
  } catch (err) {
    alert("Lỗi xóa!");
  }
};

const formatPrice = (p) => new Intl.NumberFormat('vi-VN', { style: 'currency', currency: 'VND' }).format(p || 0);
</script>

<style scoped>
.mgmt-page { background: white; padding: 25px; border-radius: 10px; box-shadow: 0 4px 15px rgba(0,0,0,0.05); }
.header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px; border-bottom: 2px solid #f4f6f8; padding-bottom: 15px;}
.header h2 { margin: 0; color: #2c3e50; }

.table { width: 100%; border-collapse: collapse; text-align: left; }
.table th, .table td { padding: 15px; border-bottom: 1px solid #eee; vertical-align: middle; }
.table th { background: #f8f9fa; color: #34495e; font-weight: 600; text-transform: uppercase; font-size: 13px; letter-spacing: 0.5px;}
.table tr:hover { background: #fdfdfd; }

.code { font-weight: bold; color: #7f8c8d; }
.price { color: #d35400; font-weight: bold; }
.text-center { text-align: center; color: #95a5a6; font-style: italic; }
.text-green { color: #27ae60; }
.drink-thumb { width: 50px; height: 50px; object-fit: cover; border-radius: 8px; border: 1px solid #ecf0f1;}

.badge-active { background: #eafaf1; color: #27ae60; padding: 4px 10px; border-radius: 4px; font-size: 12px; font-weight: bold; border: 1px solid #27ae60;}
.badge-inactive { background: #fdedec; color: #e74c3c; padding: 4px 10px; border-radius: 4px; font-size: 12px; font-weight: bold; border: 1px solid #e74c3c;}

.btn-primary { background: #2980b9; color: white; padding: 8px 15px; border: none; border-radius: 5px; cursor: pointer; font-weight: bold; transition: 0.2s;}
.btn-primary:hover { background: #3498db; }

/* Nút thao tác */
.btn-recipe { background: #8e44ad; color: white; padding: 6px 12px; border: none; border-radius: 4px; font-weight: bold; cursor: pointer; margin-right: 5px;}
.btn-recipe:hover { background: #9b59b6; }
.btn-edit { background: #f39c12; color: white; padding: 6px 12px; border: none; border-radius: 4px; font-weight: bold; cursor: pointer; margin-right: 5px;}
.btn-edit:hover { background: #e67e22; }
.btn-delete { background: #e74c3c; color: white; padding: 6px 12px; border: none; border-radius: 4px; font-weight: bold; cursor: pointer;}
.btn-delete:hover { background: #c0392b; }
.btn-delete-sm { background: transparent; color: #e74c3c; border: none; cursor: pointer; font-size: 16px; transition: 0.2s;}
.btn-delete-sm:hover { transform: scale(1.2); }

/* Modal General */
.modal-overlay { position: fixed; top: 0; left: 0; width: 100vw; height: 100vh; background: rgba(0,0,0,0.5); display: flex; justify-content: center; align-items: center; z-index: 1000; }
.modal-content { background: white; padding: 25px; border-radius: 10px; width: 500px; max-width: 90%; box-shadow: 0 10px 30px rgba(0,0,0,0.2); }
.modal-content h3 { margin-top: 0; border-bottom: 1px solid #eee; padding-bottom: 10px; color: #2c3e50;}
.modal-header { display: flex; justify-content: space-between; align-items: center; border-bottom: 1px solid #eee; padding-bottom: 15px; margin-bottom: 15px;}
.modal-header h3 { margin: 0; border: none; padding: 0;}
.highlight { color: #d35400; }
.btn-close { background: transparent; border: none; font-size: 18px; cursor: pointer; color: #7f8c8d; transition: 0.2s;}
.btn-close:hover { transform: scale(1.2); color: #e74c3c;}

/* Form CSS */
.form-grid { display: flex; flex-direction: column; gap: 15px; }
.form-row { display: flex; gap: 15px; }
.flex-1 { flex: 1; } .flex-2 { flex: 2; }
.full-width { width: 100%; }
.form-group label { font-size: 13px; color: #7f8c8d; margin-bottom: 5px; font-weight: bold; display: block;}
.form-group input[type="text"], .form-group input[type="number"], .form-group select { width: 100%; padding: 10px; border: 1px solid #bdc3c7; border-radius: 5px; box-sizing: border-box; font-family: inherit; outline: none;}
.form-group input:focus, .form-group select:focus { border-color: #3498db; }
.required { color: #e74c3c; }

.modal-actions { display: flex; justify-content: flex-end; gap: 10px; margin-top: 20px; }
.btn-cancel { background: #95a5a6; color: white; border: none; padding: 10px 15px; border-radius: 5px; cursor: pointer; font-weight: bold;}
.btn-save { background: #27ae60; color: white; border: none; padding: 10px 20px; border-radius: 5px; font-weight: bold; cursor: pointer; }

/* RECIPE MODAL SPECIFIC */
.recipe-modal { width: 600px; }
.recipe-table { margin-bottom: 20px; }
.recipe-table th { background: #fdfaf4; color: #e67e22; font-size: 12px;}
.add-recipe-box { background: #f4f6f8; padding: 15px; border-radius: 8px; border: 1px dashed #bdc3c7;}
.add-recipe-box h4 { margin: 0 0 10px 0; font-size: 13px; color: #2c3e50; text-transform: uppercase;}
.recipe-form { display: flex; gap: 10px; align-items: center;}
.relative { position: relative; }
.unit-hint { position: absolute; right: 10px; top: 10px; font-size: 13px; color: #7f8c8d; font-weight: bold; pointer-events: none;}
</style>