<template>
  <div class="mgmt-page review-mgmt">
    <div class="header">
      <h2>⭐ Quản Lý Đánh Giá (Reviews)</h2>
      <div class="header-stats">
        Tổng số: <b>{{ reviews.length }}</b> đánh giá
      </div>
    </div>

    <table class="table">
      <thead>
        <tr>
          <th>ID</th>
          <th>Khách Hàng</th>
          <th>Đồ Uống</th>
          <th>Đánh Giá</th>
          <th>Bình Luận</th>
          <th>Ngày</th>
          <th>Thao tác</th>
        </tr>
      </thead>
      <tbody>
        <tr v-if="reviews.length === 0">
          <td colspan="7" class="text-center">Chưa có đánh giá nào trên hệ thống.</td>
        </tr>
        <tr v-for="r in reviews" :key="r.reviewId">
          <td class="code">#{{ r.reviewId }}</td>
          <td>
            <b>{{ r.user?.fullName || 'Khách vãng lai' }}</b>
          </td>
          <td><b class="drink-name">{{ r.drink?.drinkName }}</b></td>
          <td>
            <div class="r-rating">
              <span v-for="star in r.rating" :key="star">⭐</span>
              <span v-for="unstar in (5 - r.rating)" :key="'u'+unstar" class="u-star">⭐</span>
              <span class="rating-num">({{ r.rating }})</span>
            </div>
          </td>
          <td>
            <div class="review-content" :title="r.content">
              {{ r.content || '(Chỉ chấm điểm, không bình luận)' }}
            </div>
          </td>
          <td>{{ formatDate(r.createdAt) }}</td>
          <td>
            <button class="btn-delete" @click="deleteReview(r.reviewId)">🗑️ Xóa</button>
          </td>
        </tr>
      </tbody>
    </table>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue';
import api from '../../services/api';

const reviews = ref([]);

onMounted(() => {
  fetchReviews();
});

const fetchReviews = async () => {
  try {
    const res = await api.get('/reviews');
    // Đảo ngược mảng để bình luận mới nhất lên đầu
    reviews.value = (res.data.data || []).reverse();
  } catch (err) {
    console.error("Lỗi lấy danh sách review:", err);
  }
};

const deleteReview = async (id) => {
  if (confirm("Bạn có chắc chắn muốn XÓA đánh giá này không?")) {
    try {
      await api.delete(`/reviews/${id}`);
      fetchReviews();
    } catch (err) {
      alert("Lỗi khi xóa: " + (err.response?.data?.message || ""));
    }
  }
};

const formatDate = (dateString) => {
  if (!dateString) return '';
  const date = new Date(dateString);
  return date.toLocaleDateString('vi-VN');
};
</script>

<style scoped>
.review-mgmt { background: white; padding: 25px; border-radius: 10px; box-shadow: 0 4px 15px rgba(0,0,0,0.05); }
.header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px; border-bottom: 2px solid #f4f6f8; padding-bottom: 15px;}
.header h2 { margin: 0; color: #2c3e50; }
.header-stats { background: #fdfaf4; padding: 8px 15px; border-radius: 20px; color: #d35400; font-size: 14px;}
.header-stats b { font-size: 16px; }

.table { width: 100%; border-collapse: collapse; text-align: left; }
.table th, .table td { padding: 15px; border-bottom: 1px solid #eee; vertical-align: middle;}
.table th { background: #f8f9fa; color: #34495e; font-weight: 600; text-transform: uppercase; font-size: 13px;}
.table tr:hover { background: #fdfdfd; }

.code { font-weight: bold; color: #7f8c8d; }
.drink-name { color: #2980b9; }
.text-center { text-align: center; color: #95a5a6; font-style: italic; }

.r-rating { color: #f1c40f; font-size: 12px;}
.u-star { opacity: 0.2; }
.rating-num { color: #7f8c8d; font-weight: bold; margin-left: 5px; font-size: 13px;}

.review-content { font-size: 13px; color: #34495e; line-height: 1.4; max-width: 250px; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; background: #f8f9fa; padding: 5px 10px; border-radius: 5px;}

.btn-delete { background: #e74c3c; color: white; border: none; padding: 8px 12px; border-radius: 6px; cursor: pointer; font-weight: bold; transition: 0.2s;}
.btn-delete:hover { background: #c0392b; }
</style>