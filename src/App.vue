<template>
  <main class="h-screen w-screen bg-black">
    <!-- Warning Overlay -->
    <div
      v-if="warningVisible"
      class="fixed inset-0 z-50 flex flex-col justify-center items-center bg-black"
    >
      <div class="border border-red-500 rounded-sm p-8 flex flex-col items-center gap-4 max-w-md text-center">
        <h1 class="text-red-500 text-2xl font-bold">⚠️ Warning!</h1>
        <p class="text-white text-lg">You left the exam tab!</p>
        <p class="text-yellow-400">Strike {{ strikes }} of 3</p>
        <p class="text-white text-sm">Leaving the exam again will result in automatic disqualification.</p>
        <button
          class="border border-white px-6 py-2 text-white rounded-sm hover:bg-white hover:text-black"
          @click="dismissWarning"
        >
          Return to Exam
        </button>
      </div>
    </div>

    <!-- Disqualified Screen -->
    <div
      v-if="isDisqualified"
      class="flex flex-col justify-center items-center h-full gap-4"
    >
      <div class="border border-red-500 rounded-sm p-8 flex flex-col items-center gap-4 max-w-md text-center">
        <h1 class="text-red-500 text-3xl font-bold">🚫 Disqualified</h1>
        <p class="text-white text-lg">You have been disqualified from the exam.</p>
        <p class="text-gray-400 text-sm">You exceeded the maximum number of violations. Please contact your instructor.</p>
      </div>
    </div>

    <!-- Login Screen -->
    <div
      class="flex flex-col justify-center items-center gap-3 h-full"
      v-if="!isShowExam && !isDisqualified"
    >
      <h1 class="text-red-500 md:text-xl">{{ info }}</h1>
      <div
        class="border border-white rounded-sm w-75 md:w-125 h-50 bg-white/50 flex flex-col justify-center items-center gap-2"
      >
        <h1 class="text-white md:text-lg">Final Examination</h1>
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
    <div
      class="flex flex-col justify-center items-center gap-3 h-full"
      v-if="isShowExam && !isDisqualified"
    >
      <div class="flex flex-col justify-center h-full w-full">
        <div class="flex justify-between items-center gap-2 py-3 px-2">
          <p class="text-lg text-white mt-2">
            Time remaining: {{ formattedTime }}
          </p>
          <div class="flex items-center gap-3">
            <span class="text-yellow-400 text-sm" v-if="strikes > 0">
              ⚠️ Strikes: {{ strikes }}/3
            </span>
            <button
              class="border border-white px-2 py-2 text-white rounded-sm hover:bg-white hover:text-black"
              @click="timeUp"
            >
              Done Exam
            </button>
          </div>
        </div>
        <iframe
          :src="pdfUrl + '#toolbar=0'"
          class="size-full"
          frameborder="0"
        ></iframe>
      </div>
    </div>
  </main>
</template>

<script setup>
import { ref, computed, onMounted, onBeforeUnmount } from "vue";

// ── Constants ────────────────────────────────────────────────
const EXAM_CODE = "123456x";
const EXAM_DURATION_MS = 3600 * 1000;
const MAX_STRIKES = 3;

// ── State ────────────────────────────────────────────────────
const pdfUrl = ref("/file/IM-Final.pdf");
const isShowExam = ref(false);
const isDisqualified = ref(false);
const timeLeft = ref(0);
const info = ref("Click 'Start' to begin the exam!");
const code = ref("");
const strikes = ref(0);
const warningVisible = ref(false);
const violationReason = ref("You left the exam tab!");

let timer = null;

// ── Computed ─────────────────────────────────────────────────
const formattedTime = computed(() => {
  const hours = Math.floor(timeLeft.value / 3600);
  const minutes = Math.floor((timeLeft.value % 3600) / 60);
  const seconds = timeLeft.value % 60;
  return `${String(hours).padStart(2, "0")}:${String(minutes).padStart(2, "0")}:${String(seconds).padStart(2, "0")}`;
});

// ── Fullscreen ───────────────────────────────────────────────
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

// ── Visibility / Alt-Tab detection ──────────────────────────
function onVisibilityChange() {
  if (document.hidden && isShowExam.value) {
    recordViolation("You switched tabs or windows!");
  }
}

// ── Strike / Disqualify ──────────────────────────────────────
function recordViolation(reason) {
  if (!isShowExam.value || warningVisible.value || isDisqualified.value) return;

  strikes.value++;
  violationReason.value = reason;
  localStorage.setItem("examStrikes", strikes.value);
  console.warn(`[Proctor] Violation #${strikes.value}: ${reason}`);

  if (strikes.value >= MAX_STRIKES) {
    disqualify();
  } else {
    warningVisible.value = true;
    localStorage.setItem("examWarning", "true");
  }
}

function disqualify() {
  clearInterval(timer);
  isShowExam.value = false;
  warningVisible.value = false;
  isDisqualified.value = true;
  unregisterProctoring();
  document.exitFullscreen?.();
  localStorage.removeItem("examWarning");
  localStorage.setItem("examDisqualified", "true");
  localStorage.setItem("examDone", "true");
}

function dismissWarning() {
  warningVisible.value = false;
  localStorage.removeItem("examWarning");
  enterFullscreen();
}

// ── Proctoring listeners ─────────────────────────────────────
function blockSelect(e) {
  if (isShowExam.value) e.preventDefault();
}

function blockCopy(e) {
  if (isShowExam.value) e.preventDefault();
}

function registerProctoring() {
  document.addEventListener("visibilitychange", onVisibilityChange);
  document.addEventListener("fullscreenchange", onFullscreenChange);
  window.addEventListener("blur", onWindowBlur);
  document.addEventListener("selectstart", blockSelect);
  document.addEventListener("copy", blockCopy);
}

function unregisterProctoring() {
  document.removeEventListener("visibilitychange", onVisibilityChange);
  document.removeEventListener("fullscreenchange", onFullscreenChange);
  window.removeEventListener("blur", onWindowBlur);
  document.removeEventListener("selectstart", blockSelect);
  document.removeEventListener("copy", blockCopy);
}

// ── Block right-click ────────────────────────────────────────
function blockEvent(e) {
  if (isShowExam.value) e.preventDefault();
}

// ── Block shortcuts ──────────────────────────────────────────
function blockShortcuts(e) {
  if (!isShowExam.value) return;

  const key = e.key.toLowerCase();
  const blocked =
    (e.ctrlKey && ["t", "n", "w", "r", "c", "a", "s", "p", "u", "escape"].includes(key)) ||
    e.key === "F5"  ||
    e.key === "F12" ||
    (e.ctrlKey && e.shiftKey && ["r", "i", "j", "c"].includes(key)) ||
    (e.altKey && e.key === "F4");

  if (blocked) e.preventDefault();
}

// ── Blur = focus lost to another tab/window ──────────────────
function onWindowBlur() {
  if (!isShowExam.value || isDisqualified.value || warningVisible.value) return;
  recordViolation("You switched to another tab or window!");
}

// ── Timer ────────────────────────────────────────────────────
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
  localStorage.removeItem("examWarning");
  localStorage.setItem("examDone", "true");
}

// ── Toggle / Start exam ──────────────────────────────────────
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

// ── Lifecycle ────────────────────────────────────────────────
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

        const hadWarning = localStorage.getItem("examWarning") === "true";
        if (hadWarning) {
          warningVisible.value = true;
        } else {
          enterFullscreen();
        }
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