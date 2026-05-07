<template>
  <main class="h-screen w-screen bg-black overflow-auto">

    <!-- Warning Overlay -->
    <div v-if="warningVisible" class="fixed inset-0 z-50 flex flex-col justify-center items-center bg-black">
      <div class="border border-red-500 rounded-sm p-8 flex flex-col items-center gap-4 max-w-md text-center">
        <h1 class="text-red-500 text-2xl font-bold">⚠️ Warning!</h1>
        <p class="text-white text-lg">You left the exam tab!</p>
        <p class="text-yellow-400">Strike {{ strikes }} of 3</p>
        <p class="text-white text-sm">Leaving the exam again will result in automatic disqualification.</p>
        <button class="border border-white px-6 py-2 text-white rounded-sm hover:bg-white hover:text-black" @click="dismissWarning">
          Return to Exam
        </button>
      </div>
    </div>

    <!-- ─── ADMIN PANEL ─────────────────────────────────────── -->
    <div v-if="currentView === 'admin'" class="min-h-screen bg-black p-6">

      <!-- Admin Login -->
      <div v-if="!isAdminLoggedIn" class="flex flex-col justify-center items-center h-screen gap-4">
        <div class="border border-white rounded-sm w-80 bg-white/10 flex flex-col items-center gap-4 p-6">
          <h1 class="text-white text-xl font-bold">🔐 Admin Panel</h1>
          <p class="text-gray-400 text-sm">Enter admin password to continue.</p>
          <input
            class="border border-white w-full p-2 rounded-sm text-white focus:outline-0"
            type="password"
            placeholder="Admin password"
            v-model="adminPassword"
            @keyup.enter="loginAdmin"
          />
          <p v-if="adminError" class="text-red-400 text-sm">{{ adminError }}</p>
          <div class="flex gap-2 w-full">
            <button class="flex-1 border border-white py-2 text-white rounded-sm hover:bg-white hover:text-black text-sm" @click="loginAdmin">
              Login
            </button>
            <button class="flex-1 border border-gray-600 py-2 text-gray-400 rounded-sm hover:bg-gray-800 text-sm" @click="currentView = 'exam'">
              Cancel
            </button>
          </div>
        </div>
      </div>

      <!-- Admin Dashboard -->
      <div v-else>
        <div class="flex justify-between items-center mb-6">
          <div>
            <h1 class="text-white text-2xl font-bold">Admin Panel</h1>
            <p class="text-gray-400 text-sm mt-1">Manage exam sessions and student records</p>
          </div>
          <button class="border border-gray-600 px-4 py-2 text-gray-400 rounded-sm hover:bg-gray-800 text-sm" @click="logoutAdmin">
            Logout
          </button>
        </div>

        <!-- Stats Row -->
        <div class="grid grid-cols-2 md:grid-cols-4 gap-3 mb-6">
          <div class="bg-white/5 border border-white/10 rounded-sm p-4">
            <p class="text-gray-400 text-xs mb-1">Total Students</p>
            <p class="text-white text-2xl font-bold">{{ students.length }}</p>
          </div>
          <div class="bg-white/5 border border-white/10 rounded-sm p-4">
            <p class="text-gray-400 text-xs mb-1">Disqualified</p>
            <p class="text-red-400 text-2xl font-bold">{{ students.filter(s => s.disqualified).length }}</p>
          </div>
          <div class="bg-white/5 border border-white/10 rounded-sm p-4">
            <p class="text-gray-400 text-xs mb-1">Exam Done</p>
            <p class="text-green-400 text-2xl font-bold">{{ students.filter(s => s.examDone).length }}</p>
          </div>
          <div class="bg-white/5 border border-white/10 rounded-sm p-4">
            <p class="text-gray-400 text-xs mb-1">In Progress</p>
            <p class="text-yellow-400 text-2xl font-bold">{{ students.filter(s => !s.examDone && s.hasSession).length }}</p>
          </div>
        </div>

        <!-- Current Device Session -->
        <div class="border border-white/20 rounded-sm mb-6">
          <div class="flex justify-between items-center px-4 py-3 border-b border-white/10">
            <div>
              <h2 class="text-white font-semibold">Current Device Session</h2>
              <p class="text-gray-400 text-xs mt-0.5">This browser's localStorage data</p>
            </div>
            <div class="flex gap-2">
              <button
                class="bg-yellow-500/10 border border-yellow-500/30 text-yellow-400 px-3 py-1.5 rounded-sm text-xs hover:bg-yellow-500/20"
                @click="resetCurrentDevice"
              >
                Reset This Device
              </button>
              <button
                class="bg-white/5 border border-white/20 text-gray-400 px-3 py-1.5 rounded-sm text-xs hover:bg-white/10"
                @click="loadStudents"
              >
                ↻ Refresh
              </button>
            </div>
          </div>

          <div class="p-4">
            <div class="grid grid-cols-2 md:grid-cols-4 gap-3">
              <div class="bg-black/30 rounded-sm p-3">
                <p class="text-gray-500 text-xs mb-1">Status</p>
                <span :class="currentSession.disqualified ? 'text-red-400' : currentSession.examDone ? 'text-green-400' : currentSession.hasSession ? 'text-yellow-400' : 'text-gray-400'" class="text-sm font-medium">
                  {{ currentSession.disqualified ? 'Disqualified' : currentSession.examDone ? 'Completed' : currentSession.hasSession ? 'In Progress' : 'Not Started' }}
                </span>
              </div>
              <div class="bg-black/30 rounded-sm p-3">
                <p class="text-gray-500 text-xs mb-1">Strikes</p>
                <p class="text-white text-sm font-medium">{{ currentSession.strikes }} / 3</p>
              </div>
              <div class="bg-black/30 rounded-sm p-3">
                <p class="text-gray-500 text-xs mb-1">Time Remaining</p>
                <p class="text-white text-sm font-medium">{{ currentSession.timeRemaining }}</p>
              </div>
              <div class="bg-black/30 rounded-sm p-3">
                <p class="text-gray-500 text-xs mb-1">End Time</p>
                <p class="text-white text-sm font-medium">{{ currentSession.endTime }}</p>
              </div>
            </div>
          </div>
        </div>

        <!-- Actions -->
        <div class="border border-white/20 rounded-sm mb-6">
          <div class="px-4 py-3 border-b border-white/10">
            <h2 class="text-white font-semibold">Quick Actions</h2>
            <p class="text-gray-400 text-xs mt-0.5">These actions apply to the current device only</p>
          </div>
          <div class="p-4 flex flex-wrap gap-3">
            <button
              class="bg-green-500/10 border border-green-500/30 text-green-400 px-4 py-2 rounded-sm text-sm hover:bg-green-500/20"
              @click="resetDisqualification"
            >
              ✅ Clear Disqualification
            </button>
            <button
              class="bg-blue-500/10 border border-blue-500/30 text-blue-400 px-4 py-2 rounded-sm text-sm hover:bg-blue-500/20"
              @click="resetStrikes"
            >
              🔄 Reset Strikes Only
            </button>
            <button
              class="bg-yellow-500/10 border border-yellow-500/30 text-yellow-400 px-4 py-2 rounded-sm text-sm hover:bg-yellow-500/20"
              @click="resetSession"
            >
              ⏱ Reset Exam Session
            </button>
            <button
              class="bg-red-500/10 border border-red-500/30 text-red-400 px-4 py-2 rounded-sm text-sm hover:bg-red-500/20"
              @click="confirmFullReset"
            >
              🗑 Full Reset
            </button>
          </div>
        </div>

        <!-- Confirm Full Reset Modal -->
        <div v-if="showConfirmReset" class="fixed inset-0 z-50 flex items-center justify-center bg-black/80">
          <div class="border border-red-500 rounded-sm p-6 flex flex-col items-center gap-4 max-w-sm text-center bg-black">
            <h2 class="text-red-400 text-lg font-bold">⚠️ Confirm Full Reset</h2>
            <p class="text-white text-sm">This will clear ALL exam data for this device including strikes, disqualification, and session. This cannot be undone.</p>
            <div class="flex gap-3 w-full">
              <button class="flex-1 bg-red-500/20 border border-red-500/40 text-red-400 py-2 rounded-sm text-sm hover:bg-red-500/30" @click="fullReset">
                Yes, Reset
              </button>
              <button class="flex-1 border border-white/20 text-gray-400 py-2 rounded-sm text-sm hover:bg-white/5" @click="showConfirmReset = false">
                Cancel
              </button>
            </div>
          </div>
        </div>

        <!-- Log -->
        <div class="border border-white/20 rounded-sm">
          <div class="flex justify-between items-center px-4 py-3 border-b border-white/10">
            <div>
              <h2 class="text-white font-semibold">Action Log</h2>
              <p class="text-gray-400 text-xs mt-0.5">Recent admin actions this session</p>
            </div>
            <button class="text-gray-600 text-xs hover:text-gray-400" @click="actionLog = []">Clear</button>
          </div>
          <div class="p-4 max-h-40 overflow-y-auto">
            <p v-if="actionLog.length === 0" class="text-gray-600 text-sm">No actions yet.</p>
            <div v-for="(entry, i) in [...actionLog].reverse()" :key="i" class="flex gap-3 items-start py-1.5 border-b border-white/5 last:border-0">
              <span class="text-gray-600 text-xs mt-0.5 shrink-0">{{ entry.time }}</span>
              <span class="text-gray-300 text-sm">{{ entry.message }}</span>
            </div>
          </div>
        </div>

      </div>
    </div>

    <!-- ─── EXAM VIEWS ───────────────────────────────────────── -->
    <div v-if="currentView === 'exam'" class="h-full">

      <!-- Disqualified Screen -->
      <div v-if="isDisqualified" class="flex flex-col justify-center items-center h-full gap-4">
        <div class="border border-red-500 rounded-sm p-8 flex flex-col items-center gap-4 max-w-md text-center">
          <h1 class="text-red-500 text-3xl font-bold">🚫 Disqualified</h1>
          <p class="text-white text-lg">You have been disqualified from the exam.</p>
          <p class="text-gray-400 text-sm">You exceeded the maximum number of violations. Please contact your instructor.</p>
        </div>
      </div>

      <!-- Login Screen -->
      <div class="flex flex-col justify-center items-center gap-3 h-full" v-if="!isShowExam && !isDisqualified">
        <h1 class="text-red-500 md:text-xl">{{ info }}</h1>
        <div class="border border-white rounded-sm w-75 md:w-125 h-50 bg-white/50 flex flex-col justify-center items-center gap-2">
          <!-- Triple-click title to open admin -->
          <h1
            class="text-white md:text-lg select-none cursor-default"
            @click="handleTitleClick"
          >
            Final Examination
          </h1>
          <input
            class="border border-white w-3/5 p-2 rounded-sm text-black focus:outline-0"
            type="password"
            placeholder="Enter code: "
            v-model.lazy="code"
            @keyup.enter="toggleExam"
          />
          <button
            class="border border-white w-3/5 py-4 text-white rounded-sm hover:bg-white hover:text-black"
            @click="toggleExam"
          >
            Start Exam
          </button>
        </div>
      </div>

      <!-- Exam Screen -->
      <div class="flex flex-col justify-center items-center gap-3 h-full" v-if="isShowExam && !isDisqualified">
        <div class="flex flex-col justify-center h-full w-full">
          <div class="flex justify-between items-center gap-2 py-3 px-2">
            <p class="text-lg text-white mt-2">Time remaining: {{ formattedTime }}</p>
            <div class="flex items-center gap-3">
              <span class="text-yellow-400 text-sm" v-if="strikes > 0">⚠️ Strikes: {{ strikes }}/3</span>
              <button
                class="border border-white px-2 py-2 text-white rounded-sm hover:bg-white hover:text-black"
                @click="timeUp"
              >
                Done Exam
              </button>
            </div>
          </div>
          <iframe :src="pdfUrl + '#toolbar=0'" class="size-full" frameborder="0"></iframe>
        </div>
      </div>

    </div>
  </main>
</template>

<script setup>
import { ref, computed, onMounted, onBeforeUnmount } from "vue";

// ── Constants ────────────────────────────────────────────────
const EXAM_CODE = "123456x";
const ADMIN_PASSWORD = "jet@2025";
const EXAM_DURATION_MS = 3600 * 1000;
const MAX_STRIKES = 3;

// ── View ─────────────────────────────────────────────────────
const currentView = ref("exam"); // 'exam' | 'admin'
const titleClickCount = ref(0);
let titleClickTimer = null;

function handleTitleClick() {
  titleClickCount.value++;
  clearTimeout(titleClickTimer);
  titleClickTimer = setTimeout(() => { titleClickCount.value = 0; }, 600);
  if (titleClickCount.value >= 3) {
    titleClickCount.value = 0;
    currentView.value = "admin";
    loadStudents();
  }
}

// ── Admin state ───────────────────────────────────────────────
const isAdminLoggedIn = ref(false);
const adminPassword = ref("");
const adminError = ref("");
const actionLog = ref([]);
const showConfirmReset = ref(false);
const students = ref([]);

const currentSession = computed(() => {
  const endTime = localStorage.getItem("examEndTime");
  const remaining = endTime ? Math.max(0, Math.floor((endTime - Date.now()) / 1000)) : null;
  const h = remaining !== null ? String(Math.floor(remaining / 3600)).padStart(2, "0") : "--";
  const m = remaining !== null ? String(Math.floor((remaining % 3600) / 60)).padStart(2, "0") : "--";
  const s = remaining !== null ? String(remaining % 60).padStart(2, "0") : "--";

  return {
    disqualified: localStorage.getItem("examDisqualified") === "true",
    examDone: localStorage.getItem("examDone") === "true",
    hasSession: !!localStorage.getItem("examEndTime"),
    strikes: parseInt(localStorage.getItem("examStrikes") || "0"),
    timeRemaining: remaining !== null ? `${h}:${m}:${s}` : "N/A",
    endTime: endTime ? new Date(parseInt(endTime)).toLocaleTimeString() : "N/A",
  };
});

function loginAdmin() {
  if (adminPassword.value === ADMIN_PASSWORD) {
    isAdminLoggedIn.value = true;
    adminError.value = "";
    adminPassword.value = "";
    loadStudents();
    logAction("Admin logged in");
  } else {
    adminError.value = "Incorrect password.";
  }
}

function logoutAdmin() {
  isAdminLoggedIn.value = false;
  currentView.value = "exam";
}

function logAction(message) {
  actionLog.value.push({
    time: new Date().toLocaleTimeString(),
    message,
  });
}

function loadStudents() {
  // Since all data is per-device localStorage, we show the current device's session
  students.value = [currentSession.value].filter(s => s.hasSession || s.disqualified);
}

function resetDisqualification() {
  localStorage.setItem("examDisqualified", "false");
  isDisqualified.value = false;
  logAction("Cleared disqualification for current device");
  loadStudents();
}

function resetStrikes() {
  localStorage.setItem("examStrikes", "0");
  strikes.value = 0;
  logAction("Reset strikes to 0 for current device");
  loadStudents();
}

function resetSession() {
  localStorage.removeItem("examEndTime");
  localStorage.setItem("examDone", "false");
  logAction("Reset exam session for current device");
  loadStudents();
}

function resetCurrentDevice() {
  resetDisqualification();
  resetStrikes();
  resetSession();
  logAction("Full reset for current device via Reset This Device button");
}

function confirmFullReset() {
  showConfirmReset.value = true;
}

function fullReset() {
  localStorage.removeItem("examStrikes");
  localStorage.removeItem("examDisqualified");
  localStorage.removeItem("examEndTime");
  localStorage.removeItem("examDone");
  strikes.value = 0;
  isDisqualified.value = false;
  isShowExam.value = false;
  timeLeft.value = 0;
  clearInterval(timer);
  showConfirmReset.value = false;
  logAction("Full reset performed");
  loadStudents();
}

// ── Exam State ────────────────────────────────────────────────
const pdfUrl = ref("/file/IM-Midterm.pdf");
const isShowExam = ref(false);
const isDisqualified = ref(false);
const timeLeft = ref(0);
const info = ref("Click 'Start' to begin the exam!");
const code = ref("");
const strikes = ref(0);
const warningVisible = ref(false);

let timer = null;

// ── Computed ──────────────────────────────────────────────────
const formattedTime = computed(() => {
  const hours = Math.floor(timeLeft.value / 3600);
  const minutes = Math.floor((timeLeft.value % 3600) / 60);
  const seconds = timeLeft.value % 60;
  return `${String(hours).padStart(2, "0")}:${String(minutes).padStart(2, "0")}:${String(seconds).padStart(2, "0")}`;
});

// ── Fullscreen ─────────────────────────────────────────────────
function enterFullscreen() {
  const el = document.documentElement;
  if (el.requestFullscreen) el.requestFullscreen();
  else if (el.webkitRequestFullscreen) el.webkitRequestFullscreen();
}

function onFullscreenChange() {
  if (!document.fullscreenElement && isShowExam.value) {
    recordViolation("You exited fullscreen!");
  }
}

// ── Visibility / Alt-Tab ──────────────────────────────────────
function onVisibilityChange() {
  if (document.hidden && isShowExam.value) {
    recordViolation("You switched tabs or windows!");
  }
}

// ── Strike / Disqualify ───────────────────────────────────────
function recordViolation(reason) {
  if (!isShowExam.value || warningVisible.value || isDisqualified.value) return;
  strikes.value++;
  localStorage.setItem("examStrikes", strikes.value);
  console.warn(`[Proctor] Violation #${strikes.value}: ${reason}`);
  if (strikes.value >= MAX_STRIKES) {
    disqualify();
  } else {
    warningVisible.value = true;
  }
}

function disqualify() {
  clearInterval(timer);
  isShowExam.value = false;
  warningVisible.value = false;
  isDisqualified.value = true;
  unregisterProctoring();
  document.exitFullscreen?.();
  localStorage.setItem("examDisqualified", "true");
  localStorage.setItem("examDone", "true");
}

function dismissWarning() {
  warningVisible.value = false;
  enterFullscreen();
}

// ── Proctoring ─────────────────────────────────────────────────
function registerProctoring() {
  document.addEventListener("visibilitychange", onVisibilityChange);
  document.addEventListener("fullscreenchange", onFullscreenChange);
}

function unregisterProctoring() {
  document.removeEventListener("visibilitychange", onVisibilityChange);
  document.removeEventListener("fullscreenchange", onFullscreenChange);
}

// ── Block right-click ──────────────────────────────────────────
function blockEvent(e) {
  if (isShowExam.value) e.preventDefault();
}

// ── Block shortcuts ────────────────────────────────────────────
function blockShortcuts(e) {
  if (!isShowExam.value) return;
  const key = e.key.toLowerCase();
  const blocked =
    (e.ctrlKey && ["t", "n", "w", "r", "c"].includes(key)) ||
    e.key === "F5" ||
    (e.ctrlKey && e.shiftKey && key === "r") ||
    (e.altKey && e.key === "F4");
  if (blocked) e.preventDefault();
}

// ── Timer ──────────────────────────────────────────────────────
function startTimer() {
  timer = setInterval(() => {
    const endTime = localStorage.getItem("examEndTime");
    const remaining = Math.floor((endTime - Date.now()) / 1000);
    if (remaining > 0) {
      timeLeft.value = remaining;
    } else {
      clearInterval(timer);
      timeUp();
    }
  }, 1000);
}

function timeUp() {
  timeLeft.value = 0;
  clearInterval(timer);
  isShowExam.value = false;
  warningVisible.value = false;
  unregisterProctoring();
  document.exitFullscreen?.();
  localStorage.setItem("examDone", "true");
}

// ── Start exam ─────────────────────────────────────────────────
function toggleExam() {
  if (isDisqualified.value) {
    info.value = "You are disqualified. Contact your instructor.";
    return;
  }
  if (!code.value) {
    info.value = "No code provided, Please enter code!";
    return;
  }
  if (code.value.toLowerCase() !== EXAM_CODE) {
    info.value = "Code is invalid";
    return;
  }

  isShowExam.value = true;
  code.value = "";

  const endTime = Date.now() + EXAM_DURATION_MS;
  localStorage.setItem("examEndTime", endTime);
  localStorage.setItem("examDone", "false");

  startTimer();
  enterFullscreen();
  registerProctoring();
}

// ── Lifecycle ──────────────────────────────────────────────────
onMounted(() => {
  strikes.value = parseInt(localStorage.getItem("examStrikes") || "0");
  isDisqualified.value = localStorage.getItem("examDisqualified") === "true";

  document.addEventListener("contextmenu", blockEvent);
  document.addEventListener("keydown", blockShortcuts);

  if (!isDisqualified.value) {
    const examDone = localStorage.getItem("examDone");
    const endTime = localStorage.getItem("examEndTime");
    if (endTime && examDone === "false") {
      const remaining = Math.floor((endTime - Date.now()) / 1000);
      if (remaining > 0) {
        isShowExam.value = true;
        timeLeft.value = remaining;
        startTimer();
        registerProctoring();
      } else {
        timeUp();
      }
    }
  }
});

onBeforeUnmount(() => {
  unregisterProctoring();
  document.removeEventListener("contextmenu", blockEvent);
  document.removeEventListener("keydown", blockShortcuts);
  clearInterval(timer);
});
</script>