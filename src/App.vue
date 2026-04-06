<template>
  <main class="h-screen w-screen bg-black">
    <div
      class="flex flex-col justify-center items-center gap-3 h-full"
      v-if="!isShowExam"
    >
      <h1 class="text-red-500 md:text-xl">{{ info }}</h1>
      <div
        class="border border-white rounded-sm w-75 md:w-125 h-50 bg-white/50 flex flex-col justify-center items-center gap-2"
      >
        <h1 class="text-white md:text-lg">Information Management</h1>
        <template v-if="!isButtonDisable">
          <button
            class="border border-white w-3/5 py-4 text-white rounded-sm hover:bg-white hover:text-black"
            :disabled="isButtonDisable"
            @click="toggleExam"
          >
            Start Exam
          </button>
        </template>
        <template v-else>
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
        </template>
      </div>
    </div>
    <div
      class="flex flex-col justify-center items-center gap-3 h-full"
      v-if="isShowExam"
    >
      <div class="flex flex-col justify-center h-full w-full">
        <div class="flex justify-between items-center gap-2 py-3 px-2">
          <p class="text-lg text-white mt-2">
            Time remaining: {{ formattedTime }}
          </p>
          <button
            class="border border-white px-2 py-2 text-white rounded-sm hover:bg-white hover:text-black"
            @click="timeUp"
          >
            Done Exam
          </button>
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

<script>
export default {
  data() {
    return {
      pdfUrl: "/file/IM-Midterm.pdf", //IM-Midterm.pdf IM-Sample.pdf
      isShowExam: false,
      timeLeft: 0,
      timer: null,
      isButtonDisable: false,
      info: "Click “Start” to begin the exam!",
      code: "",
    };
  },
  mounted() {
    const examInfo = localStorage.getItem("examInfo");
    const examDone = localStorage.getItem("examDone");
    const endTime = localStorage.getItem("examEndTime");

    if (examDone === "true") {
      this.isButtonDisable = true;
      this.info = examInfo;
    } else if (endTime) {
      const remaining = Math.floor((endTime - Date.now()) / 1000);

      if (remaining > 0) {
        this.isShowExam = true;
        this.timeLeft = remaining;
        this.startTimer();
      } else {
        this.timeUp();
      }
    }
  },
  methods: {
    toggleExam() {
      if (this.isButtonDisable && this.code.toLowerCase() === "") {
        this.info = "No code provided, Please enter code!";
        return;
      }

      if (this.isButtonDisable && this.code.toLowerCase() !== "cc104z") {
        this.info = "Code is invalid";
        return;
      }

      this.isShowExam = true;
      this.code = "";

      const endTime = Date.now() + 3600 * 1000; // 1 hour 3600
      localStorage.setItem("examInfo", "Click “Start” to begin the exam!");
      localStorage.setItem("examEndTime", endTime);
      localStorage.setItem("examDone", "false");

      this.startTimer();
    },
    startTimer() {
      this.timer = setInterval(() => {
        const endTime = localStorage.getItem("examEndTime");
        const remaining = Math.floor((endTime - Date.now()) / 1000);

        if (remaining > 0) {
          this.timeLeft = remaining;
        } else {
          clearInterval(this.timer);
          this.timeUp();
        }
      }, 1000);
    },
    timeUp() {
      this.timeLeft = 0;
      clearInterval(this.timer);
      this.isShowExam = false;
      this.isButtonDisable = true;

      localStorage.setItem(
        "examInfo",
        "Please enter the code and click “Start Exam.”",
      );

      this.info = "Please enter the code and click “Start Exam.”";
      localStorage.setItem("examDone", "true");
    },
  },
  computed: {
    formattedTime() {
      const hours = Math.floor(this.timeLeft / 3600);
      const minutes = Math.floor((this.timeLeft % 3600) / 60);
      const seconds = this.timeLeft % 60;
      return `${hours.toString().padStart(2, "0")}:${minutes
        .toString()
        .padStart(2, "0")}:${seconds.toString().padStart(2, "0")}`;
    },
  },
};
</script>
