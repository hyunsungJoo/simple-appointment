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

const GOOGLE_WEB_APP_URL = 'https://script.google.com/macros/s/AKfycby6LE5wPbJvsxo8Pl962by6Hq4iJqsIOiPXem7C1hBnQY7QrKWZGzhxKtx8m0jW2SU/exec';

const userName = ref('');
const userDept = ref('');
const userPhone = ref('');

const selectedDate = ref(null);
const selectedTime = ref(null);
const availableDates = ref([]);
const bookedSlots = ref([]); 
const globalLoading = ref(true);
const submitLoading = ref(false);

const showModal = ref(false);
const nameInput = ref(null);

const timeSlots = ['1:00 - 1:30', '1:30 - 2:00', '2:00 - 2:30', '2:30 - 3:00', '3:30 - 4:00'];

const isSlotBooked = (date, time) => {
  return bookedSlots.value.some(slot => slot.date === date && slot.time === time);
};

const openModal = () => {
  showModal.value = true;
  nextTick(() => {
    if (nameInput.value) nameInput.value.focus();
  });
};

const closeModal = () => {
  if (submitLoading.value) return;
  showModal.value = false;
};

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
/* 🛠️ 전체적으로 모든 수치를 기존 대비 약 50% 축소 세팅 */
.survey-wrapper {
  background-color: #444444;
  min-height: 100vh;
  min-height: 100dvh;
  width: 100%;
  padding: 40px 20px 120px 20px; /* 상하좌우 여백 축소 */
  box-sizing: border-box;
}

.content-wrapper { 
  max-width: 1000px; /* 메인 영역 최대 너비 대폭 축소 */
  margin: 0 auto; 
  width: 100%; 
}

.survey-header { 
  text-align: center; 
  margin-bottom: 40px; /* 마진 축소 */
}

/* 폰트 크기 반으로 축소 */
.main-title { 
  font-size: 2.8rem; 
  font-weight: 800; 
  color: #ffffff; 
  margin: 0 0 10px 0; 
  letter-spacing: -1px; 
}

.sub-title-1 { 
  font-size: 1.4rem; 
  font-weight: 600; 
  color: rgba(255, 255, 255, 0.8); 
  margin: 0; 
}

.section { 
  margin-bottom: 40px; 
  width: 100%; 
}

.section-title { 
  margin-bottom: 20px; 
  text-align: center; 
}

.section-title h3 { 
  font-size: 1.5rem; 
  font-weight: 700; 
  color: #ffffff; 
  margin: 0; 
}

.date-flex-container, .time-flex-container { 
  display: flex; 
  justify-content: center; 
  align-items: center; 
  flex-wrap: wrap; 
  gap: 15px; /* 간격 축소 */
  width: 100%; 
}

/* 🛠️ 카드 크기 및 내부 패딩 50% 축소 */
.rating-item {
  display: flex; 
  flex-direction: column; 
  align-items: center; 
  justify-content: center;
  border: 3px solid #333333; /* 선 두께 살짝 축소 */
  border-radius: 20px; /* 둥글기 최적화 */
  cursor: pointer; 
  transition: all 0.25s ease;
  background-color: #ffffff; 
  color: #1a202c; 
  box-shadow: 0 6px 20px rgba(0, 0, 0, 0.3);
  outline: none; 
  box-sizing: border-box; 
  
  width: 180px; /* 280px -> 180px로 가로 축소 */
  padding: 25px 15px; /* 내부 패딩 반으로 축소 */
}

/* 시간 카드 크기 축소 */
.time-flex-container .rating-item { 
  width: 150px; /* 240px -> 150px로 축소 */
  padding: 20px 10px; 
}

.time-text { 
  font-size: 1.5rem; /* 2.6rem -> 1.5rem */
  font-weight: 800; 
}

@media (hover: hover) { 
  .rating-item:hover:not(:disabled) { 
    border-color: #42b883; 
    background-color: #f4fbf8; 
    transform: translateY(-6px); /* 팝업 높이 조절 */
  } 
}

.rating-item.active { 
  border-color: #42b883; 
  background-color: #42b883; 
  color: #ffffff; 
}

/* 예약 마감 스타일 */
.rating-item.is-booked {
  background-color: #555555 !important; 
  border-color: #222222 !important; 
  color: #888888 !important;
  cursor: not-allowed !important; 
  box-shadow: none !important; 
  transform: none !important;
}

.booked-badge { 
  margin-top: 6px; 
  font-size: 0.9rem; 
  font-weight: bold; 
  background-color: #ff4d4f; 
  color: white; 
  padding: 2px 8px; 
  border-radius: 6px; 
}

.emoji { 
  font-size: 3.5rem; /* 6rem -> 3.5rem 아이콘 축소 */
  margin-bottom: 10px; 
}

.content-box { 
  display: flex; 
  flex-direction: column; 
  align-items: center; 
}

.text { 
  font-size: 1.4rem; /* 2.4rem -> 1.4rem */
  font-weight: bold; 
  margin-bottom: 4px; 
}

.score { 
  font-size: 1.1rem; /* 1.8rem -> 1.1rem */
  opacity: 0.6; 
}

.rating-item.active .score { opacity: 0.9; }
.loading-status { text-align: center; color: white; font-size: 1.6rem; padding: 60px 0; }

/* 🛠️ 하단 고정 바 크기 콤팩트화 */
.footer-bar { 
  position: fixed; 
  bottom: 0; left: 0; 
  width: 100%; 
  background-color: #222222; 
  border-top: 3px solid #333333; 
  box-shadow: 0 -10px 30px rgba(0, 0, 0, 0.4); 
  z-index: 100; 
  transform: translateY(100%); 
  transition: transform 0.3s ease-in-out; 
}

.footer-bar.active { transform: translateY(0); }

.footer-content { 
  max-width: 1000px; 
  margin: 0 auto; 
  display: flex; 
  justify-content: space-between; 
  align-items: center; 
  padding: 15px 25px; /* 바 두께 슬림하게 조절 */
}

.selection-info { display: flex; flex-direction: column; align-items: flex-start; color: #ffffff; }
.datetime-summary { font-size: 1.4rem; display: flex; gap: 10px; }
.final-date { font-weight: 800; color: #42b883; }
.final-time { font-weight: 800; color: #ffffff; }

/* 하단 버튼 축소 */
.submit-btn { 
  padding: 12px 35px; 
  background-color: #42b883; 
  color: white; 
  font-size: 1.3rem; 
  font-weight: 800; 
  border: none; 
  border-radius: 30px; 
  cursor: pointer; 
  box-shadow: 0 4px 10px rgba(66, 184, 131, 0.3); 
  transition: all 0.2s; 
}
.submit-btn:hover:not(:disabled) { background-color: #339d6c; transform: scale(1.03); }
.submit-btn:disabled { background-color: #555555; color: #888888; cursor: not-allowed; box-shadow: none; }

/* 🛠️ 정보 입력 팝업창(모달) 크기 슬림화 */
.modal-overlay {
  position: fixed; top: 0; left: 0;
  width: 100vw; height: 100vh;
  background-color: rgba(0, 0, 0, 0.7);
  display: flex; justify-content: center; align-items: center;
  z-index: 999;
}

.modal-card {
  background-color: #ffffff;
  border: 3px solid #333333;
  border-radius: 20px;
  width: 400px; /* 550px -> 400px 로 축소 */
  max-width: 90%;
  padding: 25px; /* 패딩 축소 */
  box-shadow: 0 12px 35px rgba(0, 0, 0, 0.5);
  box-sizing: border-box;
}

.modal-header { text-align: center; margin-bottom: 20px; }
.modal-header h3 { font-size: 1.6rem; font-weight: 800; color: #1a202c; margin: 0 0 4px 0; }
.modal-header p { font-size: 1.2rem; font-weight: 700; color: #42b883; margin: 0; }
.modal-form { display: flex; flex-direction: column; gap: 12px; margin-bottom: 25px; }
.input-block { display: flex; flex-direction: column; text-align: left; }
.input-block label { font-size: 1.1rem; font-weight: 800; color: #333333; margin-bottom: 4px; }

.input-block input {
  font-size: 1.3rem; 
  font-weight: 700;
  border: 2px solid #e2e8f0;
  border-radius: 10px;
  padding: 8px 12px; /* 인풋창 두께 슬림화 */
  outline: none;
  color: #1a202c;
}
.input-block input:focus { border-color: #42b883; }
.modal-buttons { display: flex; gap: 10px; }

.cancel-btn {
  flex: 1; padding: 12px; background-color: #eee; color: #555;
  font-size: 1.2rem; font-weight: 800; border: none; border-radius: 30px; cursor: pointer;
}
.confirm-btn {
  flex: 2; padding: 12px; background-color: #42b883; color: white;
  font-size: 1.2rem; font-weight: 800; border: none; border-radius: 30px; cursor: pointer;
}

/* 모바일 모드 최적화 (모바일은 원래 컴팩트하므로 비율 유지) */
@media (max-width: 768px) {
  .survey-wrapper { padding: 25px 15px 120px 15px; }
  .main-title { font-size: 2.2rem; }
  .sub-title-1 { font-size: 1.1rem; }
  .section-title h3 { font-size: 1.3rem; }
  .date-flex-container, .time-flex-container { flex-direction: column; align-items: center; gap: 10px; }
  .rating-item, .time-flex-container .rating-item { width: 100%; padding: 15px; flex-direction: row; justify-content: flex-start; gap: 15px; border-radius: 12px; }
  .time-flex-container .rating-item { justify-content: center; }
  .emoji { font-size: 2.2rem; margin-bottom: 0; }
  .content-box { align-items: flex-start; }
  .text { font-size: 1.3rem; margin-bottom: 0; }
  .time-text { font-size: 1.3rem; }
  .score { font-size: 1.1rem; }
  .footer-content { flex-direction: column; gap: 12px; padding: 15px; }
  .selection-info { font-size: 1.2rem; align-items: center; text-align: center; }
  .submit-btn { width: 100%; padding: 12px; font-size: 1.3rem; }
}

.fade-enter-active, .fade-leave-active { transition: opacity 0.4s ease, transform 0.4s ease; }
.fade-enter-from, .fade-leave-to { opacity: 0; transform: translateY(20px); }
.pop-enter-active, .pop-leave-active { transition: opacity 0.3s ease; }
.pop-enter-active .modal-card, .pop-leave-active .modal-card { transition: transform 0.3s cubic-bezier(0.34, 1.56, 0.64, 1); }
  .pop-enter-from, .pop-leave-to { opacity: 0; }
.pop-enter-from .modal-card, .pop-leave-to .modal-card { transform: scale(0.9); }
</style>

<style>
html, body, #app { margin: 0 !important; padding: 0 !important; height: 100% !important; width: 100% !important; background-color: #444444 !important; display: block !important; max-width: none !important; }
</style>