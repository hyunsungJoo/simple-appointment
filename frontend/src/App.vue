<template>
  <div class="survey-wrapper">
    <div class="content-wrapper">
      
      <header class="survey-header">
        <h1 class="main-title">💆 사내 마사지 예약</h1>
        <h2 class="sub-title-1">원하시는 날짜와 시간을 선택하시면 예약을 진행할 수 있습니다.</h2>
      </header>

      <div v-if="globalLoading" class="loading-status">예약 데이터를 불러오는 중입니다...</div>

      <div v-else>
        <section class="section">
          <div class="section-title">
            <h3>📅 날짜를 선택해 주세요</h3>
          </div>
          
          <div class="date-flex-container">
            <button
              v-for="date in availableDates"
              :key="date"
              :class="['rating-item', { active: selectedDate === date }]"
              @click="selectDate(date)"
            >
              <div class="emoji">📆</div>
              <div class="content-box">
                <div class="text">{{ formatDate(date) }}</div>
                <div class="score">금요일</div>
              </div>
            </button>
          </div>
        </section>

        <transition name="fade">
          <section class="section" v-if="selectedDate">
            <div class="section-title">
              <h3>⏰ 시간을 선택해 주세요 ({{ formatDate(selectedDate) }})</h3>
            </div>
            
            <div class="time-flex-container">
              <button
                v-for="time in timeSlots"
                :key="time"
                :class="['rating-item', { active: selectedTime === time }, { 'is-booked': isSlotBooked(selectedDate, time) }]"
                :disabled="isSlotBooked(selectedDate, time)"
                @click="selectTime(time)"
              >
                <div class="time-text">{{ time }}</div>
                <div v-if="isSlotBooked(selectedDate, time)" class="booked-badge">예약마감</div>
              </button>
            </div>
          </section>
        </transition>
      </div>

      <div class="footer-bar" :class="{ active: selectedDate && selectedTime }">
        <div class="footer-content">
          <div class="selection-info">
            <div v-if="selectedDate && selectedTime" class="datetime-summary">
              <span class="final-date">{{ formatDate(selectedDate) }} (금)</span>
              <span class="final-time">{{ selectedTime }}</span>
            </div>
          </div>
          <button 
            class="submit-btn" 
            :disabled="!selectedDate || !selectedTime"
            @click="openModal"
          >
            예약하기
          </button>
        </div>
      </div>

      <transition name="pop">
        <div v-if="showModal" class="modal-overlay" @click.self="closeModal">
          <div class="modal-card">
            <div class="modal-header">
              <h3>👤 예약자 정보 입력</h3>
              <p>{{ formatDate(selectedDate) }} (금) {{ selectedTime }} 예약</p>
            </div>
            
            <div class="modal-form">
              <div class="input-block">
                <label>이름</label>
                <input type="text" v-model="userName" placeholder="홍길동" ref="nameInput" />
              </div>
              <div class="input-block">
                <label>부서</label>
                <input type="text" v-model="userDept" placeholder="인사팀 / 개발팀" />
              </div>
              <div class="input-block">
                <label>연락처</label>
                <input type="tel" v-model="userPhone" placeholder="010-1234-5678" />
              </div>
            </div>

            <div class="modal-buttons">
              <button class="cancel-btn" @click="closeModal">취소</button>
              <button 
                class="confirm-btn" 
                :disabled="!userName.trim() || !userDept.trim() || !userPhone.trim() || submitLoading"
                @click="submitReservation"
              >
                {{ submitLoading ? '등록 중...' : '예약 확정하기' }}
              </button>
            </div>
          </div>
        </div>
      </transition>

    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, nextTick } from 'vue';
import axios from 'axios';

// 제공해주신 구글 웹 앱 URL 주소 그대로 유지
const GOOGLE_WEB_APP_URL = 'https://script.google.com/macros/s/AKfycby6LE5wPbJvsxo8Pl962by6Hq4iJqsIOiPXem7C1hBnQY7QrKWZGzhxKtx8m0jW2SU/exec';

// 유저 입력 정보 상태값
const userName = ref('');
const userDept = ref('');
const userPhone = ref('');

// 시스템 상태값
const selectedDate = ref(null);
const selectedTime = ref(null);
const availableDates = ref([]);
const bookedSlots = ref([]); 
const globalLoading = ref(true);
const submitLoading = ref(false);

// 팝업창(모달) 노출 여부
const showModal = ref(false);
const nameInput = ref(null);

const timeSlots = ['1:00 - 1:30', '1:30 - 2:00', '2:00 - 2:30', '2:30 - 3:00', '3:30 - 4:00'];

const isSlotBooked = (date, time) => {
  return bookedSlots.value.some(slot => slot.date === date && slot.time === time);
};

// 팝업창 열기/닫기 제어
const openModal = () => {
  showModal.value = true;
  // 팝업 열리면 자동으로 이름 입력창에 포커스 주기
  nextTick(() => {
    if (nameInput.value) nameInput.value.focus();
  });
};

const closeModal = () => {
  if (submitLoading.value) return; // 등록 중일 땐 닫기 방지
  showModal.value = false;
};

// 구글 시트에서 예약 현황 조회
const fetchReservations = async () => {
  try {
    globalLoading.value = true;
    const response = await axios.get(GOOGLE_WEB_APP_URL);
    bookedSlots.value = response.data;
  } catch (error) {
    console.error('예약 데이터 로드 실패:', error);
  } finally {
    globalLoading.value = false;
  }
};

// 예약 등록 (최종 확정)
const submitReservation = async () => {
  if (!selectedDate.value || !selectedTime.value || !userName.value.trim() || !userDept.value.trim() || !userPhone.value.trim()) return;
  
  try {
    submitLoading.value = true;
    const payload = {
      name: userName.value.trim(),
      dept: userDept.value.trim(),
      phone: userPhone.value.trim(),
      date: selectedDate.value,
      time: selectedTime.value
    };

    await axios.post(GOOGLE_WEB_APP_URL, JSON.stringify(payload), {
      headers: { 'Content-Type': 'text/plain' }
    });

    alert(`🎉 예약이 성공적으로 등록되었습니다!\n\n예약자: ${userName.value}\n날짜: ${formatDate(selectedDate.value)}\n시간: ${selectedTime.value}`);
    
    // 성공 시 모달 닫고 데이터 초기화
    showModal.value = false;
    userName.value = '';
    userDept.value = '';
    userPhone.value = '';
    selectedDate.value = null;
    selectedTime.value = null;
    
    await fetchReservations();

  } catch (error) {
    alert('예약 등록에 실패했습니다. 다시 시도해 주세요.');
    console.error(error);
  } finally {
    submitLoading.value = false;
  }
};

const calculateNextMonthFridays = () => {
  const today = new Date();
  let nextMonth = today.getMonth() + 1;
  let year = today.getFullYear();
  if (nextMonth > 11) { nextMonth = 0; year += 1; }
  const fridays = [];
  let date = new Date(year, nextMonth, 1);
  while (date.getMonth() === nextMonth) {
    if (date.getDay() === 5) {
      const formattedDate = `${date.getFullYear()}-${String(date.getMonth() + 1).padStart(2, '0')}-${String(date.getDate()).padStart(2, '0')}`;
      fridays.push(formattedDate);
    }
    date.setDate(date.getDate() + 1);
  }
  availableDates.value = fridays;
};

const formatDate = (dateString) => {
  if (!dateString) return '';
  const date = new Date(dateString);
  return `${date.getMonth() + 1}월 ${date.getDate()}일`;
};

const selectDate = (date) => { selectedDate.value = date; selectedTime.value = null; };
const selectTime = (time) => { selectedTime.value = time; };

onMounted(async () => {
  calculateNextMonthFridays();
  await fetchReservations();
});
</script>

<style scoped>
/* 기존 테마 및 레이아웃 완벽 유지 */
.survey-wrapper {
  background-color: #444444;
  min-height: 100vh;
  min-height: 100dvh;
  width: 100%;
  padding: 60px 40px 180px 40px;
  box-sizing: border-box;
}

.content-wrapper { max-width: 1600px; margin: 0 auto; width: 100%; }
.survey-header { text-align: center; margin-bottom: 80px; }
.main-title { font-size: 5.5rem; font-weight: 800; color: #ffffff; margin: 0 0 20px 0; letter-spacing: -1.5px; }
.sub-title-1 { font-size: 2.8rem; font-weight: 600; color: rgba(255, 255, 255, 0.8); margin: 0; }
.section { margin-bottom: 70px; width: 100%; }
.section-title { margin-bottom: 35px; text-align: center; }
.section-title h3 { font-size: 2.8rem; font-weight: 700; color: #ffffff; margin: 0; }

/* 날짜 및 시간 영역 레이아웃 */
.date-flex-container, .time-flex-container { display: flex; justify-content: center; align-items: center; flex-wrap: wrap; gap: 30px; width: 100%; }

.rating-item {
  display: flex; flex-direction: column; align-items: center; justify-content: center;
  border: 4px solid #333333; border-radius: 32px; cursor: pointer; transition: all 0.25s ease;
  background-color: #ffffff; color: #1a202c; box-shadow: 0 12px 35px rgba(0, 0, 0, 0.35);
  outline: none; box-sizing: border-box; width: 280px; padding: 50px 25px; 
}
.time-flex-container .rating-item { width: 240px; padding: 40px 20px; }
.time-text { font-size: 2.6rem; font-weight: 800; }

@media (hover: hover) { .rating-item:hover:not(:disabled) { border-color: #42b883; background-color: #f4fbf8; transform: translateY(-12px); } }
.rating-item.active { border-color: #42b883; background-color: #42b883; color: #ffffff; }

/* 예약 마감 슬롯 스타일 */
.rating-item.is-booked {
  background-color: #555555 !important; border-color: #222222 !important; color: #888888 !important;
  cursor: not-allowed !important; box-shadow: none !important; transform: none !important;
}
.booked-badge { margin-top: 10px; font-size: 1.4rem; font-weight: bold; background-color: #ff4d4f; color: white; padding: 4px 10px; border-radius: 10px; }

.emoji { font-size: 6rem; margin-bottom: 20px; }
.content-box { display: flex; flex-direction: column; align-items: center; }
.text { font-size: 2.4rem; font-weight: bold; margin-bottom: 8px; }
.score { font-size: 1.8rem; opacity: 0.6; }
.rating-item.active .score { opacity: 0.9; }
.loading-status { text-align: center; color: white; font-size: 2.5rem; padding: 100px 0; }

/* 🌟 팝업창 (모달 레이어) 디자인 추가 */
.modal-overlay {
  position: fixed;
  top: 0; left: 0;
  width: 100vw; height: 100vh;
  background-color: rgba(0, 0, 0, 0.7); /* 배경 어둡게 암전 효과 */
  display: flex; justify-content: center; align-items: center;
  z-index: 999;
}

.modal-card {
  background-color: #ffffff;
  border: 4px solid #333333;
  border-radius: 32px;
  width: 550px;
  max-width: 90%;
  padding: 40px;
  box-shadow: 0 20px 50px rgba(0, 0, 0, 0.5);
  box-sizing: border-box;
}

.modal-header {
  text-align: center;
  margin-bottom: 30px;
}

.modal-header h3 {
  font-size: 2.6rem;
  font-weight: 800;
  color: #1a202c;
  margin: 0 0 8px 0;
}

.modal-header p {
  font-size: 1.8rem;
  font-weight: 700;
  color: #42b883;
  margin: 0;
}

.modal-form {
  display: flex;
  flex-direction: column;
  gap: 20px;
  margin-bottom: 40px;
}

.input-block {
  display: flex;
  flex-direction: column;
  text-align: left;
}

.input-block label {
  font-size: 1.6rem;
  font-weight: 800;
  color: #333333;
  margin-bottom: 8px;
}

.input-block input {
  font-size: 2rem;
  font-weight: 700;
  border: 3px solid #e2e8f0;
  border-radius: 14px;
  padding: 12px 16px;
  outline: none;
  color: #1a202c;
  transition: border-color 0.2s;
}

.input-block input:focus {
  border-color: #42b883;
}

.modal-buttons {
  display: flex;
  gap: 15px;
}

.cancel-btn {
  flex: 1;
  padding: 18px;
  background-color: #eee;
  color: #555;
  font-size: 1.8rem;
  font-weight: 800;
  border: none;
  border-radius: 50px;
  cursor: pointer;
  transition: background-color 0.2s;
}
.cancel-btn:hover { background-color: #ddd; }

.confirm-btn {
  flex: 2;
  padding: 18px;
  background-color: #42b883;
  color: white;
  font-size: 1.8rem;
  font-weight: 800;
  border: none;
  border-radius: 50px;
  cursor: pointer;
  box-shadow: 0 5px 15px rgba(66, 184, 131, 0.3);
  transition: all 0.2s;
}
.confirm-btn:hover:not(:disabled) { background-color: #339d6c; transform: scale(1.02); }
.confirm-btn:disabled { background-color: #ccc; color: #888; cursor: not-allowed; box-shadow: none; transform: none; }

/* 하단 고정 예약 바 */
.footer-bar { position: fixed; bottom: 0; left: 0; width: 100%; background-color: #222222; border-top: 4px solid #333333; box-shadow: 0 -15px 50px rgba(0, 0, 0, 0.5); z-index: 100; transform: translateY(100%); transition: transform 0.3s ease-in-out; }
.footer-bar.active { transform: translateY(0); }
.footer-content { max-width: 1600px; margin: 0 auto; display: flex; justify-content: space-between; align-items: center; padding: 30px 40px; }
.selection-info { display: flex; flex-direction: column; gap: 8px; align-items: flex-start; color: #ffffff; }
.datetime-summary { font-size: 2.4rem; display: flex; gap: 15px; }
.final-date { font-weight: 800; color: #42b883; }
.final-time { font-weight: 800; color: #ffffff; }

.submit-btn { padding: 22px 60px; background-color: #42b883; color: white; font-size: 2.2rem; font-weight: 800; border: none; border-radius: 50px; cursor: pointer; box-shadow: 0 8px 20px rgba(66, 184, 131, 0.3); transition: all 0.2s; }
.submit-btn:hover:not(:disabled) { background-color: #339d6c; transform: scale(1.05); }
.submit-btn:disabled { background-color: #555555; color: #888888; cursor: not-allowed; box-shadow: none; }

/* 모바일 모드 정렬 최적화 */
@media (max-width: 768px) {
  .survey-wrapper { padding: 40px 20px 180px 20px; }
  .main-title { font-size: 3rem; }
  .sub-title-1 { font-size: 1.6rem; }
  .section-title h3 { font-size: 1.8rem; }
  .date-flex-container, .time-flex-container { flex-direction: column; align-items: center; gap: 15px; }
  .rating-item, .time-flex-container .rating-item { width: 100%; padding: 25px; flex-direction: row; justify-content: flex-start; gap: 20px; border-radius: 20px; }
  .time-flex-container .rating-item { justify-content: center; }
  .emoji { font-size: 3.5rem; margin-bottom: 0; }
  .content-box { align-items: flex-start; }
  .text { font-size: 1.8rem; margin-bottom: 0; }
  .time-text { font-size: 1.8rem; }
  .score { font-size: 1.4rem; }
  .footer-content { flex-direction: column; gap: 20px; padding: 20px; }
  .selection-info { font-size: 1.6rem; align-items: center; text-align: center; }
  .submit-btn { width: 100%; padding: 15px; font-size: 1.8rem; }
  .modal-card { padding: 25px 20px; }
}

/* 애니메이션 효과 */
.fade-enter-active, .fade-leave-active { transition: opacity 0.4s ease, transform 0.4s ease; }
.fade-enter-from, .fade-leave-to { opacity: 0; transform: translateY(30px); }

.pop-enter-active, .pop-leave-active { transition: opacity 0.3s ease; }
.pop-enter-active .modal-card, .pop-leave-active .modal-card { transition: transform 0.3s cubic-bezier(0.34, 1.56, 0.64, 1); }
.pop-enter-from, .pop-leave-to { opacity: 0; }
.pop-enter-from .modal-card, .pop-leave-to .modal-card { transform: scale(0.9); }
</style>

<style>
html, body, #app { margin: 0 !important; padding: 0 !important; height: 100% !important; width: 100% !important; background-color: #444444 !important; display: block !important; max-width: none !important; }
</style>