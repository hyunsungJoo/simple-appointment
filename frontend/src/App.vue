<template>
  <div class="survey-wrapper">
    <div class="content-wrapper">
      
      <header class="survey-header">
        <h1 class="main-title">💆 사내 마사지 예약</h1>
        <h2 class="sub-title-1">원하시는 메뉴를 선택하여 예약 또는 조회를 진행해 주세요.</h2>
      </header>

      <div class="tab-container">
        <button :class="{ active: currentTab === 'reserve' }" @click="currentTab = 'reserve'">📅 예약하기</button>
        <button :class="{ active: currentTab === 'lookup' }" @click="currentTab = 'lookup'">🔍 나의 예약 조회</button>
      </div>

      <div v-if="currentTab === 'reserve'">
        <div v-if="globalLoading" class="loading-status">데이터를 안전하게 동기화 중입니다. 잠시만 기다려 주세요...</div>

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
            <button class="submit-btn" :disabled="!selectedDate || !selectedTime" @click="openModal">예약하기</button>
          </div>
        </div>
      </div>

      <div v-if="currentTab === 'lookup'" class="lookup-section">
        <section class="section">
          <div class="section-title">
            <h3>🔍 등록하신 연락처를 입력해 주세요</h3>
          </div>
          
          <div class="search-box">
            <input 
              type="tel" 
              v-model="searchPhone" 
              placeholder="01012345678" 
              @keyup.enter="lookupReservations" 
            />
            <button class="search-btn" @click="lookupReservations" :disabled="lookupLoading">조회하기</button>
          </div>

          <div v-if="lookupLoading" class="loading-status">예약 명단을 매칭하는 중입니다...</div>
          
          <div v-else>
            <div v-if="lookupResults.length > 0" class="results-container">
              <div v-for="(res, index) in lookupResults" :key="index" class="rating-item result-card">
                <div class="emoji">💆</div>
                <div class="content-box">
                  <div class="text">{{ formatDate(res.date) }} (금)</div>
                  <div class="time-text">{{ res.time }}</div>
                  <div class="score">{{ res.name }}님 ({{ res.dept }})</div>
                </div>
              </div>
            </div>
            <div v-else-if="hasSearched" class="empty-msg">
              입력하신 연락처로 등록된 예약 내역이 없습니다.
            </div>
          </div>
        </section>
      </div>

      <transition name="pop">
        <div v-if="showModal" class="modal-overlay" @click.self="closeModal">
          <div class="modal-card">
            <div class="modal-header">
              <h3>👤 예약자 정보 입력</h3>
              <p>{{ formatDate(selectedDate) }} (금) {{ selectedTime }} 예약</p>
            </div>
            <div class="modal-form">
              <div class="input-block"><label>이름</label><input type="text" v-model="userName" placeholder="홍길동" ref="nameInput" /></div>
              <div class="input-block"><label>부서</label><input type="text" v-model="userDept" placeholder="경영지원팀" /></div>
              <div class="input-block"><label>휴대폰번호</label><input type="tel" v-model="userPhone" placeholder="01012345678" /></div>
            </div>
            <div class="modal-buttons">
              <button class="cancel-btn" @click="closeModal">취소</button>
              <button class="confirm-btn" :disabled="!userName.trim() || !userDept.trim() || !userPhone.trim()" @click="submitReservation">예약 확정하기</button>
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

const GOOGLE_WEB_APP_URL = 'https://script.google.com/macros/s/AKfycbw2P4xTA2RdXLoQ8Z8wF9uG13-t78LDxaWFFVebGLMYJsPSArpW6Wotw9ThA5qSScWutA/exec';

// 현재 선택된 탭 ('reserve' = 예약하기, 'lookup' = 예약조회)
const currentTab = ref('reserve');

// 조회 기능 전용 상태값
const searchPhone = ref('');
const lookupResults = ref([]);
const lookupLoading = ref(false);
const hasSearched = ref(false);

// 기존 상태값 유지
const userName = ref('');
const userDept = ref('');
const userPhone = ref('');
const selectedDate = ref(null);
const selectedTime = ref(null);
const availableDates = ref([]);
const bookedSlots = ref([]); 
const globalLoading = ref(true);
const showModal = ref(false);
const nameInput = ref(null);

const timeSlots = ['1:00 - 1:30', '1:30 - 2:00', '2:00 - 2:30', '2:30 - 3:00', '3:30 - 4:00'];

const isSlotBooked = (date, time) => {
  return bookedSlots.value.some(slot => slot.date === date && slot.time === time);
};

const openModal = () => {
  showModal.value = true;
  nextTick(() => { if (nameInput.value) nameInput.value.focus(); });
};
const closeModal = () => { showModal.value = false; };

// 전체 예약 리스트 로드
const fetchReservations = async (isSilent = false) => {
  try {
    if (!isSilent) globalLoading.value = true;
    const response = await axios.get(GOOGLE_WEB_APP_URL);
    bookedSlots.value = response.data;
  } catch (error) {
    console.error('예약 데이터 로드 실패:', error);
  } finally {
    if (!isSilent) globalLoading.value = false;
  }
};

// 💡 [신규] 특정 개인 예약 조회 함수
const lookupReservations = async () => {
  if (!searchPhone.value.trim()) return;
  
  try {
    lookupLoading.value = true;
    hasSearched.value = true;
    
    // 하이픈, 공백을 모두 제거하고 오직 '숫자만' 추출해서 파라미터로 전송
    const cleanPhone = searchPhone.value.replace(/[^0-9]/g, '');
    
    const response = await axios.get(`${GOOGLE_WEB_APP_URL}?phone=${cleanPhone}`);
    lookupResults.value = response.data;
  } catch (error) {
    console.error('예약 조회 실패:', error);
    alert('조회 중 서버 오류가 발생했습니다.');
  } finally {
    lookupLoading.value = false;
  }
};

// 신규 예약 등록
const submitReservation = () => {
  if (!selectedDate.value || !selectedTime.value || !userName.value.trim() || !userDept.value.trim() || !userPhone.value.trim()) return;
  
  const targetDate = selectedDate.value;
  const targetTime = selectedTime.value;
  const targetName = userName.value.trim();

  const payload = {
    name: targetName,
    dept: userDept.value.trim(),
    phone: userPhone.value.trim(),
    date: targetDate,
    time: targetTime
  };

  bookedSlots.value.push({ date: targetDate, time: targetTime });
  alert(`🎉 예약 신청이 접수되었습니다!\n(구글 시트에 동기화 중...)\n\n예약자: ${targetName}\n날짜: ${formatDate(targetDate)}\n시간: ${targetTime}`);
  
  showModal.value = false;
  userName.value = ''; userDept.value = ''; userPhone.value = ''; selectedDate.value = null; selectedTime.value = null;

  axios.post(GOOGLE_WEB_APP_URL, JSON.stringify(payload), {
    headers: { 'Content-Type': 'text/plain' }
  })
  .then(() => {
    fetchReservations(true);
  })
  .catch((error) => {
    alert(`⚠️ 백그라운드 전송 실패!`);
    console.error(error);
  });
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

onMounted(() => {
  calculateNextMonthFridays();
  fetchReservations(false);
});
</script>

<style scoped>
/* 💡 상단 메뉴 탭 컨테이너 디자인 */
.tab-container {
  display: flex;
  justify-content: center;
  gap: 15px;
  margin-bottom: 40px;
}
.tab-container button {
  padding: 12px 25px;
  font-size: 1.3rem;
  font-weight: 800;
  border: 3px solid #333333;
  background-color: #ffffff;
  color: #333333;
  border-radius: 16px;
  cursor: pointer;
  box-shadow: 0 6px 20px rgba(0, 0, 0, 0.2);
  transition: all 0.2s ease;
}
.tab-container button.active {
  background-color: #42b883;
  color: white;
  border-color: #42b883;
}

/* 💡 연락처 검색 박스 디자인 */
.search-box {
  display: flex;
  justify-content: center;
  gap: 12px;
  margin-bottom: 40px;
  width: 100%;
}
.search-box input {
  font-size: 1.4rem;
  font-weight: 700;
  border: 3px solid #333333;
  border-radius: 16px;
  padding: 12px 20px;
  outline: none;
  width: 320px;
  max-width: 65%;
  box-shadow: 0 6px 20px rgba(0, 0, 0, 0.3);
  background-color: white;
  color: #1a202c;
}
.search-btn {
  padding: 12px 25px;
  background-color: #42b883;
  color: white;
  font-size: 1.3rem;
  font-weight: 800;
  border: 3px solid #333333;
  border-radius: 16px;
  cursor: pointer;
  box-shadow: 0 6px 20px rgba(0, 0, 0, 0.3);
  transition: background-color 0.2s;
}
.search-btn:hover { background-color: #339d6c; }

/* 조회 결과 카드 배치 배열 */
.results-container {
  display: flex;
  justify-content: center;
  flex-wrap: wrap;
  gap: 20px;
  width: 100%;
}
.result-card {
  width: 220px !important;
  cursor: default !important;
}
.result-card:hover {
  transform: none !important;
  border-color: #333333 !important;
  background-color: white !important;
}

/* 기존 50% 축소 스타일 완벽 보존 */
.survey-wrapper { background-color: #444444; min-height: 100vh; min-height: 100dvh; width: 100%; padding: 40px 20px 120px 20px; box-sizing: border-box; }
.content-wrapper { max-width: 1000px; margin: 0 auto; width: 100%; }
.survey-header { text-align: center; margin-bottom: 40px; }
.main-title { font-size: 2.8rem; font-weight: 800; color: #ffffff; margin: 0 0 10px 0; letter-spacing: -1px; }
.sub-title-1 { font-size: 1.4rem; font-weight: 600; color: rgba(255, 255, 255, 0.8); margin: 0; }
.section { margin-bottom: 40px; width: 100%; }
.section-title { margin-bottom: 20px; text-align: center; }
.section-title h3 { font-size: 1.5rem; font-weight: 700; color: #ffffff; margin: 0; }
.date-flex-container, .time-flex-container { display: flex; justify-content: center; align-items: center; flex-wrap: wrap; gap: 15px; width: 100%; }

.rating-item {
  display: flex; flex-direction: column; align-items: center; justify-content: center;
  border: 3px solid #333333; border-radius: 20px; cursor: pointer; transition: all 0.25s ease;
  background-color: #ffffff; color: #1a202c; box-shadow: 0 6px 20px rgba(0, 0, 0, 0.3);
  outline: none; box-sizing: border-box; width: 180px; padding: 25px 15px; 
}
.time-flex-container .rating-item { width: 150px; padding: 20px 10px; }
.time-text { font-size: 1.5rem; font-weight: 800; }

@media (hover: hover) { .rating-item:hover:not(:disabled) { border-color: #42b883; background-color: #f4fbf8; transform: translateY(-6px); } }
.rating-item.active { border-color: #42b883; background-color: #42b883; color: #ffffff; }

.rating-item.is-booked {
  background-color: #555555 !important; border-color: #222222 !important; color: #888888 !important;
  cursor: not-allowed !important; box-shadow: none !important; transform: none !important;
}
.booked-badge { margin-top: 6px; font-size: 0.9rem; font-weight: bold; background-color: #ff4d4f; color: white; padding: 2px 8px; border-radius: 6px; }
.emoji { font-size: 3.5rem; margin-bottom: 10px; }
.content-box { display: flex; flex-direction: column; align-items: center; }
.text { font-size: 1.4rem; font-weight: bold; margin-bottom: 4px; }
.score { font-size: 1.1rem; opacity: 0.6; }
.loading-status { text-align: center; color: #42b883; font-size: 1.8rem; font-weight: bold; padding: 100px 0; }
.empty-msg { text-align: center; color: rgba(255, 255, 255, 0.6); font-size: 1.4rem; padding: 40px 0; font-weight: bold; }

.footer-bar { position: fixed; bottom: 0; left: 0; width: 100%; background-color: #222222; border-top: 3px solid #333333; box-shadow: 0 -10px 30px rgba(0, 0, 0, 0.4); z-index: 100; transform: translateY(100%); transition: transform 0.3s ease-in-out; }
.footer-bar.active { transform: translateY(0); }
.footer-content { max-width: 1000px; margin: 0 auto; display: flex; justify-content: space-between; align-items: center; padding: 15px 25px; }
.selection-info { display: flex; flex-direction: column; align-items: flex-start; color: #ffffff; }
.datetime-summary { font-size: 1.4rem; display: flex; gap: 10px; }
.final-date { font-weight: 800; color: #42b883; }
.final-time { font-weight: 800; color: #ffffff; }

.submit-btn { padding: 12px 35px; background-color: #42b883; color: white; font-size: 1.3rem; font-weight: 800; border: none; border-radius: 30px; cursor: pointer; box-shadow: 0 4px 10px rgba(66, 184, 131, 0.3); transition: all 0.2s; }
.submit-btn:hover:not(:disabled) { background-color: #339d6c; transform: scale(1.03); }
.submit-btn:disabled { background-color: #555555; color: #888888; cursor: not-allowed; box-shadow: none; }

.modal-overlay { position: fixed; top: 0; left: 0; width: 100vw; height: 100vh; background-color: rgba(0, 0, 0, 0.7); display: flex; justify-content: center; align-items: center; z-index: 999; }
.modal-card { background-color: #ffffff; border: 3px solid #333333; border-radius: 20px; width: 400px; max-width: 90%; padding: 25px; box-shadow: 0 12px 35px rgba(0, 0, 0, 0.5); box-sizing: border-box; }
.modal-header { text-align: center; margin-bottom: 20px; }
.modal-header h3 { font-size: 1.6rem; font-weight: 800; color: #1a202c; margin: 0 0 4px 0; }
.modal-header p { font-size: 1.2rem; font-weight: 700; color: #42b883; margin: 0; }
.modal-form { display: flex; flex-direction: column; gap: 12px; margin-bottom: 25px; }
.input-block { display: flex; flex-direction: column; text-align: left; }
.input-block label { font-size: 1.1rem; font-weight: 800; color: #333333; margin-bottom: 4px; }
.input-block input { font-size: 1.3rem; font-weight: 700; border: 2px solid #e2e8f0; border-radius: 10px; padding: 8px 12px; outline: none; color: #1a202c; }
.input-block input:focus { border-color: #42b883; }
.modal-buttons { display: flex; gap: 10px; }
.cancel-btn { flex: 1; padding: 12px; background-color: #eee; color: #555; font-size: 1.2rem; font-weight: 800; border: none; border-radius: 30px; cursor: pointer; }
.confirm-btn { flex: 2; padding: 12px; background-color: #42b883; color: white; font-size: 1.2rem; font-weight: 800; border: none; border-radius: 30px; cursor: pointer; }

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
  .modal-card { padding: 25px 20px; }
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