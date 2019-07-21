<template>
  <div id="app">
    <img src="./assets/hint_up/freecell_logo@2x.png"
      alt=""
      class="logo">
    <div class="round">#{{6384730}}</div>
    <div class="time">Time: 00:00</div>
    <div class="score">Score: 0</div>
    <img src="./assets/hint_up/undo@2x.png"
      alt=""
      class="undo"
      @click="(isWin = true, panelIsShow = true)">
    <img src="./assets/hint_up/hint@2x.png"
      alt=""
      class="hint"
      @click="(isWin = false, panelIsShow = true)">
    <img src="./assets/hint_up/Group 1293.svg"
      alt=""
      class="group">
    <div class="board">
      <div class="free-space">
        <div class="free-space-ctn"
          v-for="i in 4"
          :key="i"></div>
      </div>
      <div class="target-space">
        <div class="target-ctn"
          v-for="i in 4"
          :key="i"></div>
      </div>
      <div class="bottom-space">
        <div class="stack"
          v-for="(stack, i) in cards"
          :key="i">
          <div class="card"
            v-for="(card, j) in stack"
            :key="j">
            <img :src="getCard(card)">
          </div>
        </div>
      </div>
    </div>
    <Panel v-if="panelIsShow"
      v-model="panelIsShow"
      :isWin="isWin"></Panel>
  </div>
</template>
<script>
import shuffle from 'lodash/shuffle'
import Panel from './components/Panel'
const card = require.context('./assets/CARDS')
const sort = ['tr', 'ca', 'cc', 'pi']
function cardCal (i) {
  if (i === 0) return false
  const number = (i - 1) % 13 + 1
  const category = sort[~~((i - 1) / 13)]
  return card(`./${category}_${number}.svg`)
}

export default {
  name: 'app',
  components: {
    Panel
  },
  created () {
    this.init()
  },
  data () {
    return {
      /** @type {number[][]} */
      cards: [],
      /** @type {number[]} */
      freeSpace: [0, 0, 0, 0],
      /** @type {number[]} */
      targetSpace: [0, 0, 0, 0],
      panelIsShow: false,
      isWin: false
    }
  },
  methods: {
    init () {
      let cards = Array.from({ length: 52 }, (_, i) => (i + 1))
      cards = shuffle(cards)
      for (let i = 0; i < 4; i++) {
        this.cards.push(cards.splice(0, 7))
      }
      for (let i = 0; i < 4; i++) {
        this.cards.push(cards.splice(0, 6))
      }
      console.log(this.cards)
    },
    freeSpaceOnDragover () {

    },
    freeSpaceOnDrop () { },
    getCard (num) {
      return cardCal(num)
    }
  }
}
</script>
<style lang="stylus">
size($w, $h = $w)
  height $h
  width $w
html, body
  width 100%
  height 100%
  margin 0
  padding 0
#app
  font-family 'Segoe UI Regular', 'Avenir', Helvetica, Arial, sans-serif
  -webkit-font-smoothing antialiased
  -moz-osx-font-smoothing grayscale
  size 1920px 1080px
  background-color #0F1620
  position relative
  .logo
    size 384px 51px
    top 52px
    left 180px
    position absolute
  .round, .time, .score
    color #E6BF8D
    white-space nowrap
  .round
    size 129px 37px
    font-size 28px
    position absolute
    left 597px
    top 66px
  .time
    position absolute
    size 218px 53px
    left 1292px
    top 50px
    font-size 40px
    font-weight bold
  .score
    position absolute
    size 160px 53px
    left 1580px
    top 50px
    font-size 40px
    font-weight bold
  .undo, .hint
    position absolute
    size 50px
    cursor pointer
    right 66px
  .group
    cursor pointer
    size 52px 43px
    position absolute
    right 64px
    top 327px
  .undo
    top 143px
  .hint
    top 235px
  .free-space
    size 706px 215px
    position absolute
    top 143px
    left 180px
    display flex
    justify-content space-between
    align-items center
    .free-space-ctn
      size 154px 215px
      border-radius 18px
      display flex
      justify-content center
      align-items center
      background url('assets/hint_up/Component 2 – 32.svg')
      background-size cover
  .target-space
    size 706px 215px
    position absolute
    top 143px
    left 1034px
    display flex
    justify-content space-between
    align-items center
    .target-ctn
      display flex
      justify-content center
      align-items center
      size 154px 215px
      border-radius 18px
      border 7px solid #FCCD8D
      background-color #171D29
  .bottom-space
    position absolute
    top 421px
    left 50%
    transform translateX(-50%)
    display flex
    width 1508px
    justify-content space-between
    align-content flex-start
    margin 0 auto
    .stack
      width 152px
      min-height 215px
      display inline-block
      .card
        margin-bottom -160px
        position relative
        &:last-of-type:hover:after
          content ''
          position absolute
          top -16px
          left -16px
          size 184px 245px
          background url('./assets/hint_up/hint_up@2x.png')
          background-size contain
</style>
