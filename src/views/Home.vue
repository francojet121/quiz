<template>
<div class="bytee-quiz">
  <error-renderer :message="errorMessage" :severity="errorSeverity"
                  @close="errorMessage = ''"></error-renderer>

  <div class="quiz-info" v-if="numQuestion !== -1 && numQuestion !== questions.length">
    <div class="columns">
      <div class="quiz-question-number column col-6" :title="$t('question_number')">
        {{numQuestion + 1}} {{ $t('of') }} {{questions.length}}
      </div>
      <div v-show="!resolveResolution" :title="$t('time_left')"
           class="quiz-time-left column col-6 text-right">
        <i class="fa fa-time"></i> {{timerText}}
      </div>
    </div>
  </div>

  <start
      v-show="numQuestion === -1"
      :quiz="quiz"
      @start="startQuiz">
  </start>

  <question
      v-if="numQuestion !== -1 && numQuestion !== questions.length"
      :question.sync="activeQuestion"
      :resolve="resolveResolution"
      @answer="answer"
  >
  </question>

  <final
      v-if="numQuestion === questions.length && !resolveResolution"
      :questions="questions"
      @resolveQuiz="resolveQuiz"
      @goToQuestion="viewQuestion"
      @previousQuestion="previousQuestion"
  >
  </final>

  <result
      v-if="resolveResolution && numQuestion === questions.length"
      :questions="questions"
      @viewQuestion="viewQuestion"
  >
  </result>

  <!-- Quiz Nav, TODO MOVE -->
  <div class="quiz-nav" v-if="numQuestion !== -1 && numQuestion !== questions.length">
    <div class="columns">
      <div class="column col-8 col-md-12">
        <button
            class="btn btn-primary"
            v-if="!resolveResolution && numQuestion !== 0"
            @click="firstQuestion"
        >
          {{ $t('back_to_start') }}
        </button>
        <button
            class="btn btn-primary"
            v-if="!resolveResolution && numQuestion !== 0"
            @click="previousQuestion"
        >
          {{ $t('previous_question') }}
        </button>
        <button
            class="btn btn-primary"
            v-if="!resolveResolution && numQuestion >= 0"
            @click="markQuestion"
        >
          <span v-show="activeQuestion.marked">{{ $t('clear_flag') }}</span>
          <span v-show="!activeQuestion.marked">{{ $t('flag_question') }}</span>
        </button>
        <button
            class="btn btn-primary"
            v-if="resolveResolution"
            @click="numQuestion = questions.length"
        >
          {{ $t('back_to_results') }}
        </button>
      </div>
      <div class="column col-4 col-md-auto text-right" v-if="quiz.apiEndpoint">
        <router-link to="/suggest" target="_blank">{{ $t('suggest_question') }}</router-link>
        |
        <router-link :to="{name: 'report', params: { id: activeQuestion._id } }"
                     target="_blank">
          {{ $t('report_question') }}
        </router-link>
      </div>
    </div>
  </div>
</div>
</template>

<script>
import isEqual from 'lodash/isEqual';
import shuffle from 'lodash/shuffle';

import Start from './Start';
import Question from './Question';
import Result from './Result';
import ErrorRenderer from './parts/ErrorRenderer';
import Final from './Final';

export default {
  name: 'home',
  components: {
    Final,
    ErrorRenderer,
    Result,
    Question,
    Start,
  },
  created() {
    if (window.Quiz.customizeQuiz) {
      this.$router.push('/customize');
      return;
    }

    // Very simple load
    this.$http
      .get(this.quiz.questionsEndpoint)
      .then((response) => {
        this.prepareQuiz(response.data);
      })
      .catch((error) => {
        this.errorMessage = error;
        this.errorSeverity = 'error';
      });
  },
  methods: {
    startQuiz() {
      this.startTimer(this.quiz.duration * 60);
      this.nextQuestion();
    },

    /**
     * Prepare the quiz question
     * @param questions {Array} Questions for this Quiz
     */
    prepareQuiz(questions) {
      if (this.quiz.randomize) {
        // First shuffle the question ordering
        questions = shuffle(questions);
      }

      // Shrink questions
      if (this.quiz.questionCount && questions.length > this.quiz.questionCount) {
        questions = questions.slice(0, this.quiz.questionCount);
      }

      // Question answers
      // Shuffle questions, stupid hack TODO Improve
      questions.forEach((question, index) => {
        question.index = index;

        if (!this.quiz.randomize) {
          return;
        }

        if (question.kind === 'text') {
          return;
        }

        question.answers.forEach((answer, index) => {
          if (question.resolution.includes(index)) {
            answer.isCorrect = true;
          }
        });

        question.answers = shuffle(question.answers);
        question.resolution = [];

        question.answers.forEach((answer, index) => {
          if (answer.isCorrect) {
            question.resolution.push(index);
            delete answer.isCorrect;
          }
        });
      });


      this.questions = questions;
    },

    startTimer(duration) {
      this.timer = setInterval(() => {
        let minutes = parseInt(duration / 60, 10);
        let seconds = parseInt(duration % 60, 10);

        minutes = minutes < 10 ? `0${minutes}` : minutes;
        seconds = seconds < 10 ? `0${seconds}` : seconds;

        this.timerText = `${minutes}:${seconds}`;

        duration -= 1;

        if (duration < 0) {
          duration = 0;

          // Finish Quiz
          this.numQuestion = this.questions.length;
          this.resolveResolution = true;
          clearInterval(this.timer);
        }
      }, 1000);
    },

    answer(val) {
      // Checking done here for convenience
      this.activeQuestion.answer = val;

      if (this.activeQuestion.kind !== 'text') {
        this.activeQuestion.isCorrect = isEqual(val, this.activeQuestion.resolution);
      } else {
        // Multiple answers could be correct for text questions
        this.activeQuestion.isCorrect = this.activeQuestion.resolution.includes(val);
      }

      console.log(this.activeQuestion);

      this.nextQuestion();
    },

    // Navigation
    firstQuestion() {
      this.numQuestion = 0;

      this.activeQuestion = this.questions[this.numQuestion];
    },

    previousQuestion() {
      this.numQuestion -= 1;

      this.activeQuestion = this.questions[this.numQuestion];
    },

    nextQuestion() {
      this.numQuestion += 1;

      if (this.numQuestion === this.questions.length) {
        return;
      }

      this.activeQuestion = this.questions[this.numQuestion];
    },

    markQuestion() {
      this.activeQuestion.marked = !this.activeQuestion.marked;

      this.nextQuestion();
    },

    sendStats() {
      if (!window.Quiz.statisticEndpoint) {
        return;
      }

      // Report users result
      this.$http
        .post(`${window.Quiz.apiEndpoint}/stats`, this.questions)
        .then(() => {
        })
        .catch((error) => {
          this.errorMessage = error;
          this.errorSeverity = 'warning';
        });
    },

    // Result
    viewQuestion(index) {
      this.numQuestion = index;
      this.activeQuestion = this.questions[this.numQuestion];
    },

    resolveQuiz() {
      this.sendStats();
      this.resolveResolution = true;
      clearInterval(this.timer);
    },
  },
  data() {
    return {
      // For now
      quiz: window.Quiz,
      questions: [],

      // Simple Quiz State
      timer: null,
      timerText: '',

      numQuestion: -1,
      activeQuestion: {},

      // Results
      resolveResolution: false,

      // Error Handling
      errorMessage: '',
      errorSeverity: 'warning',
    };
  },
};
</script>

<style>
  .quiz-info {
    margin-bottom: 10px;
  }
</style>
