<template>
  <div id="app"
    ref="app">
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
          v-for="(card, i) in freeSpace"
          :key="i">
          <img v-if="card !== 0"
            :src="getCard(card.value)"
            alt="">
        </div>
      </div>
      <div class="target-space">
        <div class="target-ctn"
          v-for="(topCard, i) in targetSpace"
          :key="i">
          <img v-if="topCard !== 0"
            :src="getCard(topCard)"
            alt="">
        </div>
      </div>
      <div class="bottom-space">
        <div class="stack"
          v-for="(stack, i) in cards"
          :key="i">
          <div class="card"
            :draggable="card.draggable"
            v-for="(card, j) in stack"
            :data-value="card.value"
            :key="j">
            <img :src="getCard(card.value)">
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
import Panel from './components/Panel'
import _ from 'lodash'
import constant from './constant'
const card = require.context('./assets/CARDS')
const suits = ['tr', 'ca', 'cc', 'pi']
/**
 * @typedef CardBasic
 * @property {number} value
 * @property {boolean} draggable
 */

/**
 * @typedef PokerBasic
 * @property {number} number
 * @property {'tr'|'ca'|'cc'|'pi'} suit
 */

/** @type {[number, {number:number, suit: 'tr'|'ca'|'cc'|'pi'}][]} */
const CardTypes = _.range(1, 53).map(v => [v, { number: (v - 1) % 13 + 1, suit: suits[~~((v - 1) / 13)] }])

export default {
  name: 'app',
  components: {
    Panel
  },
  created () {
    this.init()
    window.tmpMoveToFreeSpace = this.tmpMoveToFreeSpace.bind(this)
    window.tmpMoveToStack = this.tmpMoveToStack.bind(this)
    this.tmpDragList = {
      cardsPos: {
        i: 0,
        j: 6,
        length: 1
      },
      from: 'stack'
    }
  },
  mounted () {
    this.tableResize()
    window.addEventListener('resize', () => this.tableResize())
  },
  data () {
    return {
      /** @type {CardBasic[][]} */
      cards: [],
      /** @type {CardBasic[]} */
      freeSpace: [0, 0, 0, 0],
      /** @type {number[]} */
      targetSpace: [0, 0, 0, 0],
      panelIsShow: false,
      isWin: false,
      /** @type {{cardsPos: {i: number, j:number, length:number}, from: 'stack'} | {cardsPos: {i:number}, from: 'freeSpace'}} */
      tmpDragList: null,
      memoryStack: [],
      cardType: new Map(CardTypes)
    }
  },
  computed: {
    allowMovementAmount () {
      let topFreeSpace = _.sum(this.freeSpace.map(e => +(!!e))) + 1
      let bottomFreeSpace = _.sum(this.cards.map(e => (!!e && +(!!e.length)) || 0)) + 1
      if ((topFreeSpace + bottomFreeSpace) === 3) return 1
      else if ((topFreeSpace + bottomFreeSpace) === 2) return 0
      else {
        return Math.min(topFreeSpace * bottomFreeSpace, 13)
      }
    }
  },
  methods: {
    tableResize () {
      const table = this.$refs['app']
      if (!table) return
      const clientWidth = window.innerWidth
      let scaleX = 1
      scaleX = clientWidth / (table.offsetWidth + 25)
      table.style.transform = `scale(${scaleX > 1 ? 1 : scaleX})`
      table.style.margin = '0 auto'
      table.style.transformOrigin = 'top left'
    },
    init () {
      let cards = Array.from({ length: 52 }, (_, i) => ({ value: (i + 1), draggable: false }))
      cards = _.shuffle(cards)
      for (let i = 0; i < 4; i++) {
        this.cards.push(this.cardInStackIsDraggable(cards.splice(0, 7)))
      }
      for (let i = 0; i < 4; i++) {
        this.cards.push(this.cardInStackIsDraggable(cards.splice(0, 6)))
      }
      console.log(this.cards)
    },
    /** @param {CardBasic[]} stack */
    cardInStackIsDraggable (stack = []) {
      if (!stack.length) return stack
      let prevCard = stack[stack.length - 1]
      prevCard.draggable = true
      // FIXME: reset the calculate
      if (stack.length === 1) return stack
      for (let i = stack.length - 2; i >= (this.allowMovementAmount <= stack.length ? stack.length - this.allowMovementAmount : 0); (i--)) {
        if (stack[i] === stack[i + 1].value + 1) {
          stack[i].draggable = true
        } else {
          return stack
        }
      }
      return stack
    },
    getCardColor (suit = '') {
      return ['tr', 'pi'].includes(suit) ? 'black' : 'red'
    },
    /**
     * @param {PokerBasic} card
     * @param {CardBasic[]} targetStack
     */
    stackCanDrop (card, targetStack) {
      if (targetStack.length === 0) return true
      const targetStackTopCard = this.cardType.get(targetStack[targetStack.length - 1].value)
      if (this.getCardColor(card.suit) === this.getCardColor(targetStackTopCard)) {
        return false
      } else {
        if ((card.number + 1) === targetStackTopCard.number) {
          return true
        } else {
          return false
        }
      }
    },
    freeSpaceCanDrop (targetId = 0) {
      return this.freeSpace[targetId] === 0
    },
    /**
     * @param {number} freeSpaceId
     */
    tmpMoveToFreeSpace (freeSpaceId) {
      if (this.tmpDragList.from === 'freeSpace') {
        if (this.tmpDragList.cardsPos.i === freeSpaceId) return true
        else {
          [this.freeSpace[freeSpaceId], this.freeSpace[this.tmpDragList.cardsPos.i]] = [this.freeSpace[this.tmpDragList.cardsPos.i], this.freeSpace[freeSpaceId]]
          this.pushToMemoryStack(this.tmpDragList.from, 'freeSpace', this.tmpDragList.cardsPos, { i: freeSpaceId })
          this.tmpDragList = null
          return true
        }
      } else {
        if (this.tmpDragList.cardsPos.length > 1) {
          alert(constant.THE_ACTION_IS_INVALID)
          this.tmpDragList = null
          return false
        } else {
          let { i } = this.tmpDragList.cardsPos
          this.freeSpace[freeSpaceId] = this.cards[i].pop()
          this.pushToMemoryStack(this.tmpDragList.from, 'freeSpace', this.tmpDragList.cardsPos, { i: freeSpaceId })
          this.tmpDragList = null
          return true
        }
      }
    },
    /**
     * @param {number} stackId
     */
    tmpMoveToStack (stackId) {
      if (this.tmpDragList.from === 'freeSpace') {
        // confirm the input card value is card stack last card value - 1
        this.cards[stackId].push(this.freeSpace[this.tmpDragList.cardsPos.i])
        this.freeSpace[this.tmpDragList.cardsPos.i] = 0
        this.pushToMemoryStack(this.tmpDragList.from, 'stack', this.tmpDragList.cardsPos, { i: stackId })
        this.tmpDragList = null
        return true
      } else {
        if (this.tmpDragList.cardsPos.length > this.allowMovementAmount) {
          alert(constant.THE_ACTION_IS_INVALID)
          this.tmpDragList = null
          return false
        }
        [].push.call(this.cards[stackId], ...this.cards[this.tmpDragList.cardsPos.i].splice(this.tmpDragList.cardsPos.j))
        this.pushToMemoryStack(this.tmpDragList.from, 'stack', this.tmpDragList.cardsPos, { i: stackId })
        this.tmpDragList = null
        return true
      }
    },
    scanAndMoveToTargetSpace () {

    },
    pushToMemoryStack (from, to, cardsPos, targetPos) {
      this.memoryStack.push({ from, to, fromDetail: JSON.parse(JSON.stringify(cardsPos)), toDetail: JSON.parse(JSON.stringify(targetPos)) })
    },
    recoveryFromMemoryStack () {
      // this.memoryStack
    },
    cardOnDragStart () { },
    cardOnDragEnd () { },
    freeSpaceOnDrop () { },
    getCard (num) {
      return card(`./${this.cardType.get(num).suit}_${this.cardType.get(num).number}.svg`)
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
  background linear-gradient(#0F1620 80%, rgba(#0F1620, 0.6))
  overflow-x hidden
#app
  font-family 'Segoe UI Regular', 'Avenir', Helvetica, Arial, sans-serif
  -webkit-font-smoothing antialiased
  -moz-osx-font-smoothing grayscale
  size 1920px 1080px
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
        &[draggable='true']:hover
          margin-bottom -140px
          &:not(:active):after
            content ''
            position absolute
            top -16px
            left -16px
            size 184px 245px
            background url('./assets/hint_up/hint_up@2x.png')
            background-size contain
</style>
