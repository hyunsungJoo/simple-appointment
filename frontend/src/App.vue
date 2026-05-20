<template>
  <div class="survey-wrapper">
    <div class="content-wrapper">
      
      <header class="survey-header">
        <h1 class="main-title">[EAP 웰니스 프로그램]</h1>
        <h1 class="main-title">피지컬 테라피 예약</h1>
        <h2 class="sub-title-1">원하시는 메뉴를 선택하여 예약 또는 조회를 진행해 주세요.</h2>
      </header>

      <div class="tab-container">
        <button :class="{ active: currentTab === 'reserve' }" @click="currentTab = 'reserve'">📅 예약하기</button>
        <button :class="{ active: currentTab === 'lookup' }" @click="currentTab = 'lookup'">🔍 나의 예약 조회/취소</button>
      </div>

      <div v-if="currentTab === 'reserve'">
        <div v-if="globalLoading" class="loading-status">화면을 불러오는 중입니다. 잠시만 기다려 주세요...</div>

        <div v-else>
          <section class="section">
            <div class="section-title month-select-title">
              <h3>📅 날짜를 선택해 주세요</h3>
              <div class="month-tabs">
                <button 
                  v-for="m in availableMonths" 
                  :key="m" 
                  :class="['month-btn', { active: selectedMonth === m }]"
                  @click="changeMonth(m)"
                >
                  {{ m }}월
                </button>
              </div>
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
              <div v-if="availableDates.length === 0" class="empty-msg">해당 월에는 예약 가능한 금요일이 없습니다.</div>
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
            <h3>🔍 등록하신 예약자 성명을 입력해 주세요</h3>
          </div>
          
          <div class="search-box">
            <input 
              type="text" 
              v-model="searchName" 
              placeholder="홍길동" 
              @keyup.enter="lookupReservations" 
            />
            <button class="search-btn" @click="lookupReservations" :disabled="lookupLoading">조회하기</button>
          </div>

          <div v-if="lookupLoading" class="loading-status">예약 명단을 최신화하여 확인 중입니다...</div>
          
          <div v-else>
            <div v-if="lookupResults.length > 0" class="results-container">
              <div 
                v-for="(res, index) in lookupResults" 
                :key="index" 
                :class="['rating-item', 'result-card', { 'is-cancelled-card': res.isCancelled }]"
              >
                <div class="emoji">{{ res.isCancelled ? '❌' : '💆' }}</div>
                <div class="content-box">
                  <div class="text">{{ formatDate(res.date) }} (금)</div>
                  <div class="time-text">{{ res.time }}</div>
                  <div class="score">{{ res.name }}님 ({{ res.dept }})</div>
                  
                  <div v-if="res.isCancelled" class="cancel-badge">취소완료</div>
                  <button v-else class="card-cancel-btn" @click="cancelReservation(res)">취소하기</button>
                </div>
              </div>
            </div>
            <div v-else-if="hasSearched" class="empty-msg">
              입력하신 성명으로 등록된 예약 내역이 없습니다.
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
              <div class="input-block"><label>부서</label><input type="text" v-model="userDept" placeholder="경영지원팀" ref="deptInput" /></div>
              <div class="input-block"><label>휴대폰번호</label><input type="tel" v-model="userPhone" placeholder="01012345678" ref="phoneInput" /></div>
            </div>
            <div class="modal-buttons">
              <button class="cancel-btn" @click="closeModal">취소</button>
              <button class="confirm-btn" @click="submitReservation">예약 확정하기</button>
            </div>
          </div>
        </div>
      </transition>

      <transition name="pop">
        <div v-if="customAlert.show" class="modal-overlay" style="z-index: 10000;">
          <div class="custom-alert-card">
            <div class="custom-alert-msg">{{ customAlert.message }}</div>
            <div class="custom-alert-buttons" v-if="customAlert.type !== 'loading'">
              <button v-if="customAlert.type === 'confirm'" class="cancel-btn" @click="handleAlertCancel">취소</button>
              <button class="confirm-btn" @click="handleAlertConfirm">확인</button>
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

const GOOGLE_WEB_APP_URL = 'https://script.google.com/macros/s/AKfycbwh77xrwb9ySOJm1tuly_5uVlAQmaA9i-tx1VQ5Qj6RcHyMSavE2brje_kGMmNHLUDqpg/exec';

const currentTab = ref('reserve');
const availableMonths = [6, 7, 8];
const selectedMonth = ref(6); 
const currentYear = new Date().getFullYear();

const searchName = ref('');
const lookupResults = ref([]);
const lookupLoading = ref(false);
const hasSearched = ref(false);

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
const deptInput = ref(null);
const phoneInput = ref(null);

const timeSlots = ['1:00 - 1:30', '1:30 - 2:00', '2:00 - 2:30', '2:30 - 3:00', '3:30 - 4:00'];

const customAlert = ref({ show: false, type: 'alert', message: '', resolve: null });

const openAlert = (message) => new Promise((resolve) => { customAlert.value = { show: true, type: 'alert', message, resolve }; });
const openConfirm = (message) => new Promise((resolve) => { customAlert.value = { show: true, type: 'confirm', message, resolve }; });
const openLoading = (message) => { customAlert.value = { show: true, type: 'loading', message, resolve: null }; };

// 💡 확인 버튼 클릭 시 모달도 강제로 닫고 첫 화면으로 복귀
const handleAlertConfirm = () => { 
  customAlert.value.show = false; 
  showModal.value = false; 
  currentTab.value = 'reserve'; 
  if (customAlert.value.resolve) customAlert.value.resolve(true); 
};

const handleAlertCancel = () => { 
  customAlert.value.show = false; 
  if (customAlert.value.resolve) customAlert.value.resolve(false); 
};

const isSlotBooked = (date, time) => bookedSlots.value.some(slot => slot.date === date && slot.time === time);
const openModal = () => { showModal.value = true; nextTick(() => { if (nameInput.value) nameInput.value.focus(); }); };
const closeModal = () => { showModal.value = false; };

const fetchReservations = async (isSilent = false) => {
  try {
    if (!isSilent) globalLoading.value = true;
    const response = await axios.get(`${GOOGLE_WEB_APP_URL}?t=${new Date().getTime()}`);
    bookedSlots.value = response.data;
  } catch (error) { console.error('예약 데이터 로드 실패:', error); } finally { if (!isSilent) globalLoading.value = false; }
};

const lookupReservations = async () => {
  if (!searchName.value.trim()) { await openAlert('🔍 조회하실 예약자 성명을 입력해 주세요.'); return; }
  try {
    lookupLoading.value = true; hasSearched.value = true;
    const response = await axios.get(`${GOOGLE_WEB_APP_URL}?name=${encodeURIComponent(searchName.value.trim())}&t=${new Date().getTime()}`);
    lookupResults.value = response.data.reverse();
  } catch (error) { await openAlert('조회 실패'); } finally { lookupLoading.value = false; }
};

const cancelReservation = async (res) => {
  if (!await openConfirm(`🚨 예약 취소하시겠습니까?`)) return;
  res.isCancelled = true;
  bookedSlots.value = bookedSlots.value.filter(s => !(s.date === res.date && s.time === res.time));
  await openAlert('❌ 취소되었습니다.');
  axios.post(GOOGLE_WEB_APP_URL, JSON.stringify({ action: 'cancel', name: res.name, date: res.date, time: res.time }), { headers: { 'Content-Type': 'text/plain' } })
    .then(() => fetchReservations(true)).catch(async () => { res.isCancelled = false; await openAlert(`⚠️ 통신 실패`); });
};

const submitReservation = async () => {
  if (!userName.value.trim()) { await openAlert('이름 입력 필수'); nameInput.value?.focus(); return; }
  if (!userDept.value.trim()) { await openAlert('부서 입력 필수'); deptInput.value?.focus(); return; }
  if (!userPhone.value.trim()) { await openAlert('전화번호 입력 필수'); phoneInput.value?.focus(); return; }
  
  if (bookedSlots.value.filter(s => s.name === userName.value.trim()).length >= 2) {
    await openAlert(`🚨 예약 한도 초과! (최대 2개)`); return;
  }

  openLoading('예약 가능여부를 확인중입니다.\n잠시만 기다려 주세요...');

  try {
    const response = await axios.post(GOOGLE_WEB_APP_URL, JSON.stringify({ name: userName.value.trim(), dept: userDept.value.trim(), phone: userPhone.value.trim(), date: selectedDate.value, time: selectedTime.value }), { headers: { 'Content-Type': 'text/plain' } });
    
    customAlert.value.show = false; 
    
    if (response.data.result === 'error') {
      await openAlert(`⚠️ 예약 실패:\n${response.data.message}`);
      // 💡 여기서 handleAlertConfirm이 호출되면서 자동으로 Tab이 reserve로 바뀝니다.
      fetchReservations(true);
    } else {
      bookedSlots.value.push({ date: selectedDate.value, time: selectedTime.value, name: userName.value.trim() });
      await openAlert(`🎉 예약 확정!`);
      // 💡 예약 성공 후 폼 초기화
      userName.value = ''; userDept.value = ''; userPhone.value = ''; selectedDate.value = null; selectedTime.value = null;
      lookupResults.value = []; hasSearched.value = false; fetchReservations(true);
    }
  } catch (error) { customAlert.value.show = false; await openAlert(`⚠️ 통신 오류`); }
};

const calculateFridaysForMonth = (m) => {
  const fridays = []; let date = new Date(currentYear, m - 1, 1);
  while (date.getMonth() === m - 1) {
    if (date.getDay() === 5) fridays.push(`${date.getFullYear()}-${String(date.getMonth() + 1).padStart(2, '0')}-${String(date.getDate()).padStart(2, '0')}`);
    date.setDate(date.getDate() + 1);
  }
  availableDates.value = fridays;
};

const changeMonth = (m) => { selectedMonth.value = m; selectedDate.value = null; selectedTime.value = null; calculateFridaysForMonth(m); };
const formatDate = (dateString) => { if (!dateString) return ''; const date = new Date(dateString); return `${date.getMonth() + 1}월 ${date.getDate()}일`; };
const selectDate = (date) => { selectedDate.value = date; selectedTime.value = null; };
const selectTime = (time) => { selectedTime.value = time; };

onMounted(() => { changeMonth(selectedMonth.value); fetchReservations(false); });
</script>

<style scoped>
/* 기본 스타일 유지 */
.month-select-title { display: flex; flex-direction: column; align-items: center; gap: 15px; }
.month-tabs { display: flex; gap: 10px; }
.month-btn { padding: 8px 18px; font-size: 1.2rem; font-weight: 800; border-radius: 20px; border: 2px solid #333; background-color: white; color: #333; cursor: pointer; transition: all 0.2s; box-shadow: 0 4px 10px rgba(0, 0, 0, 0.15); }
.month-btn.active { background-color: #42b883; color: white; border-color: #42b883; }
@media (min-width: 768px) { .month-select-title { flex-direction: row; justify-content: center; } }

.tab-container { display: flex; justify-content: center; gap: 15px; margin-bottom: 40px; }
.tab-container button { padding: 12px 25px; font-size: 1.3rem; font-weight: 800; border: 3px solid #333333; background-color: #ffffff; color: #333333; border-radius: 16px; cursor: pointer; box-shadow: 0 6px 20px rgba(0, 0, 0, 0.2); transition: all 0.2s ease; }
.tab-container button.active { background-color: #42b883; color: white; border-color: #42b883; }

.search-box { display: flex; justify-content: center; gap: 12px; margin-bottom: 40px; width: 100%; }
.search-box input { font-size: 1.4rem; font-weight: 700; border: 3px solid #333333; border-radius: 16px; padding: 12px 20px; outline: none; width: 320px; max-width: 65%; box-shadow: 0 6px 20px rgba(0, 0, 0, 0.3); background-color: white; color: #1a202c; }
.search-btn { padding: 12px 25px; background-color: #42b883; color: white; font-size: 1.3rem; font-weight: 800; border: 3px solid #333333; border-radius: 16px; cursor: pointer; box-shadow: 0 6px 20px rgba(0, 0, 0, 0.3); transition: background-color 0.2s; }
.search-btn:hover:not(:disabled) { background-color: #339d6c; }
.search-btn:disabled { background-color: #555555; color: #888888; cursor: not-allowed; border-color: #222222; }

.results-container { display: flex; justify-content: center; flex-wrap: wrap; gap: 20px; width: 100%; }
.result-card { width: 220px !important; cursor: default !important; }
.result-card:hover { transform: none !important; border-color: #333333 !important; background-color: white !important; }

.card-cancel-btn { margin-top: 12px; padding: 6px 16px; font-size: 1.1rem; font-weight: 800; background-color: #ffffff; color: #ff4d4f; border: 2px solid #ff4d4f; border-radius: 12px; cursor: pointer; transition: all 0.2s; }
.card-cancel-btn:hover { background-color: #fff1f0; }

.is-cancelled-card { opacity: 0.45; background-color: #f5f5f5 !important; border-color: #cccccc !important; }
.cancel-badge { margin-top: 12px; font-size: 1.1rem; font-weight: bold; color: #888888; background-color: #e8e8e8; padding: 4px 12px; border-radius: 8px; }

.survey-wrapper { background-color: #444444; min-height: 100vh; min-height: 100dvh; width: 100%; padding: 40px 20px 120px 20px; box-sizing: border-box; }
.content-wrapper { max-width: 1000px; margin: 0 auto; width: 100%; }
.survey-header { text-align: center; margin-bottom: 40px; }
.main-title { font-size: 2.8rem; font-weight: 800; color: #ffffff; margin: 0 0 10px 0; letter-spacing: -1px; }
.sub-title-1 { font-size: 1.4rem; font-weight: 600; color: rgba(255, 255, 255, 0.8); margin: 0; }
.section { margin-bottom: 40px; width: 100%; }
.section-title { margin-bottom: 20px; text-align: center; }
.section-title h3 { font-size: 1.5rem; font-weight: 700; color: #ffffff; margin: 0; }
.date-flex-container, .time-flex-container { display: flex; justify-content: center; align-items: center; flex-wrap: wrap; gap: 15px; width: 100%; }

.rating-item { display: flex; flex-direction: column; align-items: center; justify-content: center; border: 3px solid #333333; border-radius: 20px; cursor: pointer; transition: all 0.25s ease; background-color: #ffffff; color: #1a202c; box-shadow: 0 6px 20px rgba(0, 0, 0, 0.3); outline: none; box-sizing: border-box; width: 180px; padding: 25px 15px; }
.time-flex-container .rating-item { width: 150px; padding: 20px 10px; }
.time-text { font-size: 1.5rem; font-weight: 800; }

@media (hover: hover) { .rating-item:hover:not(:disabled) { border-color: #42b883; background-color: #f4fbf8; transform: translateY(-6px); } }
.rating-item.active { border-color: #42b883; background-color: #42b883; color: #ffffff; }
.rating-item.is-booked { background-color: #555555 !important; border-color: #222222 !important; color: #888888 !important; cursor: not-allowed !important; box-shadow: none !important; transform: none !important; }
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

/* 모달 기본 스타일 */
.modal-overlay { position: fixed; top: 0; left: 0; width: 100vw; height: 100vh; background-color: rgba(0, 0, 0, 0.8); display: flex; justify-content: center; align-items: center; z-index: 999; }
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

/* 💡 추가된 대형 커스텀 알림창 스타일 */
.custom-alert-card {
  background-color: #ffffff;
  border: 4px solid #333333;
  border-radius: 24px;
  width: 500px;
  max-width: 90%;
  padding: 40px;
  box-shadow: 0 15px 40px rgba(0, 0, 0, 0.6);
  text-align: center;
  box-sizing: border-box;
}
.custom-alert-msg {
  font-size: 1.8rem;
  font-weight: 800;
  color: #1a202c;
  line-height: 1.6;
  margin-bottom: 35px;
  white-space: pre-wrap; /* \n 줄바꿈 적용되도록 설정 */
}
.custom-alert-buttons { display: flex; gap: 15px; }
.custom-alert-buttons .cancel-btn { padding: 18px; font-size: 1.5rem; border-radius: 50px; }
.custom-alert-buttons .confirm-btn { padding: 18px; font-size: 1.5rem; border-radius: 50px; }

@media (max-width: 768px) {
  .survey-wrapper { padding: 25px 15px 120px 15px; }
  .main-title { font-size: 2.2rem; }
  .sub-title-1 { font-size: 1.1rem; }
  .section-title h3 { font-size: 1.3rem; }
  .month-select-title { flex-direction: column; }
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
  .custom-alert-card { padding: 30px 20px; }
  .custom-alert-msg { font-size: 1.5rem; margin-bottom: 25px; }
  .custom-alert-buttons .cancel-btn, .custom-alert-buttons .confirm-btn { padding: 14px; font-size: 1.3rem; }
}

.fade-enter-active, .fade-leave-active { transition: opacity 0.4s ease, transform 0.4s ease; }
.fade-enter-from, .fade-leave-to { opacity: 0; transform: translateY(20px); }
.pop-enter-active, .pop-leave-active { transition: opacity 0.3s ease; }
.pop-enter-active .modal-card, .pop-leave-active .modal-card,
.pop-enter-active .custom-alert-card, .pop-leave-active .custom-alert-card { transition: transform 0.3s cubic-bezier(0.34, 1.56, 0.64, 1); }
.pop-enter-from, .pop-leave-to { opacity: 0; }
.pop-enter-from .modal-card, .pop-leave-to .modal-card,
.pop-enter-from .custom-alert-card, .pop-leave-to .custom-alert-card { transform: scale(0.9); }
</style>

<style>
html, body, #app { margin: 0 !important; padding: 0 !important; height: 100% !important; width: 100% !important; background-color: #444444 !important; display: block !important; max-width: none !important; }
</style>