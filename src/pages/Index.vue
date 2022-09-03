<template>
  <div style="margin: 0;
    padding: 0;
    box-sizing: border-box;">
      <div ref="main" :style="'position: relative; margin: 0; padding: 0; height: ' + mainHeight + 'px; overflow-y: hidden; padding-left: ' + paddingLeft + 'px; padding-top: ' + paddingTop + 'px;'">
        <div :style="'font-size: 2em; position: relative; display: inline-block; zoom: ' + zoomScale + '; width: ' + divWidth + 'px; height: ' + divHeight + 'px;'">
          <div style="height: 280px; overflow-y: hidden; ">
            <div style="font-size: 2em; padding: 0.2em; padding-left: 1em;">{{ currentQuestion.question }}</div>
            <div style="font-size: 2em; padding: 0.2em; padding-left: 1em; display: flex;">
              <template :key="index" v-for="(option, index) in currentQuestion.options">
                <div>{{ optionIndex(index) }}) {{ option.value }}</div>
                <div style="width: 100px; padding-right: 1em;">
                  <q-icon v-if="correct[index]" size="80px" name="done" color="positive" />
                  <q-icon v-if="wrong[index]" size="80px" name="close" color="negative" />
                </div>
              </template>
            </div>
          </div>
          <div style="height: 800px;">
            <iframe
              ref="targetFrame"
              style="background: white;"
              width="100%" height="100%" frameBorder="0"
              :src="'/game/index.html'"
            />
          </div>
        </div>

        <div v-if="!gameStarted" :style="'position: absolute; left: 0px; top: 0px; padding-left: ' + paddingLeft + 'px;; padding-top: ' + paddingTop + 'px;'">
          <div :style="'position: relative; background: rgba(192, 192, 192, 0.5); zoom: ' + zoomScale + '; width: ' + divWidth + 'px; height: ' + divHeight + 'px;'">
            <div style="position: absolute; left: 500px; top: 150px;">
              <img src="/images/start-box.png" />
            </div>
            <div v-if="isGodotReady" @click="startGame" style="cursor: pointer; position: absolute; left: 870px; top: 480px;">
              <img src="/images/button-play.png" />
            </div>
          </div>
        </div>
      </div>
  </div>
</template>

<script>
import { defineComponent } from 'vue'

export default defineComponent({
  name: 'game',
  setup() {
    
  },
  data () {
    return {
      zoomScale: 1,
      divHeight: 1080,
      divWidth: 1920,
      paddingLeft: 0,
      paddingTop: 0,
      mainHeight: 0,
      showGameOver: false,
      selectedOption: '',
      selectedIndex: '',
      gameStarted: false,
      questionIndex: 0,
      correct: [],
      wrong: [],
      questions: [
        { question: '1 + 0', options: [{ value: 1, correct: true }, { value: 2, correct: false }, { value: 3, correct: false }, { value: 4, correct: false }], score: 10 },
        { question: '1 + 1', options: [{ value: 1, correct: false }, { value: 2, correct: true }, { value: 3, correct: false }, { value: 4, correct: false }], score: 11 },
        { question: '1 + 2', options: [{ value: 1, correct: false }, { value: 2, correct: false }, { value: 3, correct: true }, { value: 4, correct: false }], score: 12 },
        { question: '1 + 3', options: [{ value: 1, correct: false }, { value: 2, correct: false }, { value: 3, correct: false }, { value: 4, correct: true }], score: 13 }
      ],

      nextQuestionTimer: null,
      isGodotReady: false
    }
  },
  computed: {
    currentQuestion () {
      if (this.questions[this.questionIndex]) {
        return this.questions[this.questionIndex]
      } else {
        return this.questions[0]
      }
    }
  },
  methods: {
    nextQuestion () {
      this.reset()
      
      this.questionIndex++

      if (this.questionIndex === this.questions.length) {
        this.questionIndex = 0
      }

      if (this.$refs['targetFrame'] && this.$refs['targetFrame'].contentWindow) {
        this.$refs['targetFrame'].contentWindow.nextQuestion(this.currentQuestion.score)
      }
    },
    reset () {
      this.selectedIndex = ''
      this.selectedOption = ''

      for (let i = 0; i < 4; i++) {
        this.correct[i] = false
        this.wrong[i] = false
      }
    },
    optionIndex (index) {
      return String.fromCharCode(65 + index)
    },
    startGame () {
      this.gameStarted = true
      this.showGameOver = false
      this.selectedOption = ''
      if (this.$refs['targetFrame'] && this.$refs['targetFrame'].contentWindow) {
        this.$refs['targetFrame'].contentWindow.startGame()
      }
    },
    stopGame () {
      if (this.$refs['targetFrame'] && this.$refs['targetFrame'].contentWindow) {
        this.$refs['targetFrame'].contentWindow.stopGame()
      }
    },
    setDivHeight () {
      if ((this.divHeight / this.divWidth) < (window.innerHeight / window.innerWidth)) {
        this.mainHeight = window.innerHeight
        let _width = window.innerWidth

        this.zoomScale = _width / this.divWidth
        
        let _value = (window.innerHeight - this.divHeight * this.zoomScale) / 2
        this.paddingTop = Math.max(0, _value)
        this.paddingLeft = 0
      } else {
        this.mainHeight = window.innerHeight
        let _width = window.innerWidth

        this.zoomScale = window.innerHeight / this.divHeight

        let _value = (_width - this.divWidth * this.zoomScale) / 2

        this.paddingTop = 0
        this.paddingLeft = Math.max(0, _value)
      }
    },
    showStartMenu () {
      setTimeout(() => {
        this.showGameOver = true
        this.gameStarted = false
      }, 1000)
    },
    receiveMessage (event) {
      if (event.data.message === 'gameReady') {
        this.isGodotReady = true
        if (this.$refs['targetFrame'] && this.$refs['targetFrame'].contentWindow) {
          this.$refs['targetFrame'].contentWindow.nextQuestion(this.currentQuestion.score)
        }
      } else if (event.data.message === 'timesup') {
        this.showStartMenu()
      } else if (event.data.message === 'noLife') {
        this.showStartMenu()
      } else if (event.data.message === 'selectOption') {
        if (this.selectedOption === '') {
          this.selectedOption = event.data.option
          this.selectedIndex = event.data.option.charCodeAt(0) - 65

          if (this.currentQuestion.options[this.selectedIndex].correct) {
            this.correct[this.selectedIndex] = true

            if (this.$refs['targetFrame'] && this.$refs['targetFrame'].contentWindow) {
              this.$refs['targetFrame'].contentWindow.correct()
            }
          } else {
            this.wrong[this.selectedIndex] = true

            if (this.$refs['targetFrame'] && this.$refs['targetFrame'].contentWindow) {
              this.$refs['targetFrame'].contentWindow.wrong()
            }
          }

          this.nextQuestionTimer = setTimeout(() => this.nextQuestion(), 1000);
        }
      } else if (event.data.message === 'getGameSetting') {
        if (this.$refs['targetFrame'] && this.$refs['targetFrame'].contentWindow) {
          this.$refs['targetFrame'].contentWindow.setGame(this.getNoOfLife(), this.getTimeLimit())
        }
      }
    },
    getNoOfLife () {
      return 2
    },
    getTimeLimit () {
      return 45
    }
  },
  mounted () {
    window.addEventListener('resize', this.setDivHeight, false)
    window.addEventListener('message', this.receiveMessage)
    
    this.setDivHeight()
  },
  beforeDestroy () {
    window.removeEventListener('resize', this.setDivHeight)
    window.removeEventListener('message', this.receiveMessage)
  }
})
</script>
