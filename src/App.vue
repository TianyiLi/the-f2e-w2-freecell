<template>
  <div id="app"
    @contextmenu.prevent="() => (forceMoveToTargetSpace(), false)"
    ref="app">
    <img src="./assets/hint_up/freecell_logo@2x.png"
      alt=""
      class="logo">
    <div class="round">#{{roundNumber}}</div>
    <div class="time">Time: {{formatTime}}</div>
    <div class="score">Score: {{score()}}</div>
    <img src="./assets/hint_up/undo@2x.png"
      alt=""
      class="undo"
      @click.stop="recoveryFromMemoryStack()">
    <img src="./assets/hint_up/hint@2x.png"
      alt=""
      class="hint"
      @click="hintingOnClick()">
    <img src="./assets/hint_up/Group 1293.svg"
      alt=""
      @click="init()"
      class="group">
    <div class="board" @click="boardOnClick">
      <div class="free-space" @clic.stop>
        <div class="free-space-ctn"
          v-for="(card, i) in freeSpace"
          :class="hintingMap.get(card.value)"
          @dragover.prevent
          @drop.prevent="freeSpaceOnDrop($event, i)"
          @click.prevent.stop="cardOnClick('freeSpace', {i})"
          :tabindex="card.value"
          :key="i">
          <img v-if="card !== 0"
            :draggable="true"
            :data-value="card.value"
            :ref="'card' + card.value"
            @dragend.prevent="cardOnDragEnd()"
            :class="((card.value === clicked) && 'is-click') || ''"
            @dragstart.stop="cardOnDragStart($event, {i}, 'freeSpace', 'card' + card.value)"
            :src="getCard(card.value)"
            alt="">
        </div>
      </div>
      <div class="target-space" @clic.stop>
        <div class="target-ctn"
          @dragover.prevent
          @drop.prevent="targetOnDrop($event, i)"
          v-for="(topCard, i) in targetSpace"
          @click.prevent.stop="cardOnClick('target', {i})"
          :tabIndex="topCard.value"
          :key="i">
          <img v-if="topCard !== 0"
            draggable="false"
            :src="getCard(topCard)"
            alt="">
        </div>
      </div>
      <div class="bottom-space" @clic.stop>
        <div class="stack"
          :class="stack.length > 8 ? 'tight':''"
          v-for="(stack, i) in cards"
          draggable="false"
          @drop.prevent="stackOnDrop($event, i)"
          @dragover.prevent
          :key="i">
          <div class="card"
            :class="hintingMap.get(card.value)"
            :draggable="'' + card.draggable"
            :data-value="card.value"
            @dblclick.capture.stop="card.draggable ? stackCardOnDBClick(i, j) : stackCardOnDBClick(i, stack.length - 1)"
            @dragend.prevent="cardOnDragEnd()"
            @dragstart.stop="cardOnDragStart($event, {i, j}, 'stack', 'card' + card.value)"
            @click.prevent.stop="card.draggable ? cardOnClick('stack', {i, j}) : cardOnClick('stack', {i, j: stack.length - 1})"
            v-for="(card, j) in stack"
            :key="j">
            <img :ref="'card' + card.value"
              :class="((card.value === clicked) && 'is-click') || ''"
              :draggable="'' + card.draggable"
              :src="getCard(card.value)">
          </div>
        </div>
      </div>
    </div>
    <Panel v-if="panelIsShow"
      v-model="panelIsShow"
      @panel="panelOnClick"
      :isWin="isWin"></Panel>
    <div class="ghost-space">
      <div class="ghost"
        id="ghost"
        ref="ghost"></div>
    </div>
  </div>
</template>
<script>
import Panel from './components/Panel'
import _ from 'lodash'
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

/** @type {[number, PokerBasic][]} */
const CardTypes = _.range(1, 53).map(v => [v, { number: (v - 1) % 13 + 1, suit: suits[~~((v - 1) / 13)] }])

export default {
  name: 'app',
  components: {
    Panel
  },
  mounted () {
    this.init()
    window.$v = this
  },
  data () {
    return {
      /** @type {CardBasic[][]} */
      cards: [],
      currentRoundSnapShot: '',
      /** @type {CardBasic[]} */
      freeSpace: [0, 0, 0, 0],
      /** @type {number[]} */
      targetSpace: [0, 0, 0, 0],
      autoScanNeededCardMem: new Map([['*', 0]]),
      panelIsShow: false,
      isWin: false,
      /** @type {{cardsPos: {i: number, j:number, length:number}, from: 'stack'} | {cardsPos: {i:number}, from: 'freeSpace'}} */
      tmpDragList: false,
      memoryStack: [],
      cardType: new Map(CardTypes),
      status: 'start',
      roundNumber: ~~(Math.random() * 9999999),
      hintingMap: new Map(),
      clicked: -1,
      timer: -1,
      timeStamp: 0
    }
  },
  computed: {
    allowMovementAmount () {
      let topFreeSpace = _.sum(this.freeSpace.map(e => +(!e))) + 1
      let bottomFreeSpace = _.sum(this.cards.map(e => {
        return (!!e && +(!e.length)) || 0
      })) + 1
      if ((topFreeSpace + bottomFreeSpace) === 3) return 2
      else if ((topFreeSpace + bottomFreeSpace) === 2) return 1
      else {
        return Math.min(topFreeSpace * bottomFreeSpace, 13)
      }
    },
    formatTime () {
      let timeStamp = this.timeStamp
      let sec = '00' + timeStamp % 60
      let mins = '00' + ~~(timeStamp / 60)
      return `${mins.slice(-2)}:${sec.slice(-2)}`
    }
  },
  methods: {
    boardOnClick () {
      this.clicked = -1
      console.log('boardClick')
    },
    score () {
      return _.sum(this.targetSpace.map(e => (e && this.cardType.get(e).number) || 0))
    },
    init (isRestart = false) {
      this.clicked = -1
      this.freeSpace = [0, 0, 0, 0]
      this.targetSpace = [0, 0, 0, 0]
      this.tmpDragList = false
      this.memoryStack = []
      this.autoScanNeededCardMem = new Map([['*', 0]])
      this.isWin = false
      let _tmpStacks = []
      this.hintingMap = new Map()
      this.cards = Array(8).fill([{ value: 1, draggable: false }])
      this.timeStamp = 0
      clearInterval(this.timer)
      let cards = Array.from({ length: 52 }, (_, i) => ({ value: (i + 1), draggable: false }))
      cards = _.shuffle(cards)
      if (isRestart) {
        cards = JSON.parse(this.currentRoundSnapShot)
      } else {
        this.currentRoundSnapShot = JSON.stringify(cards)
        this.roundNumber = ~~(Math.random() * 9999999)
      }
      for (let i = 0; i < 4; i++) {
        _tmpStacks.push(this.cardInStackIsDraggable(cards.splice(0, 7)))
      }
      for (let i = 0; i < 4; i++) {
        _tmpStacks.push(this.cardInStackIsDraggable(cards.splice(0, 6)))
      }
      this.timer = setInterval(() => {
        this.timeStamp++
      }, 1000)
      this.cards = JSON.parse(JSON.stringify(_tmpStacks))
    },
    /** @param {CardBasic[]} stack */
    cardInStackIsDraggable (stack = []) {
      if (!stack.length) return stack
      let prevCard = stack[stack.length - 1]
      prevCard.draggable = true
      if (stack.length === 1) return stack
      for (let i = stack.length - 2; i >= (this.allowMovementAmount <= stack.length ? stack.length - this.allowMovementAmount : 0); (i--)) {
        if (this.stackCanDrop(this.cardType.get(stack[i].value), stack.slice(0, i))) {
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
     * @param {number} length
     */
    stackCanDrop (card, targetStack, length) {
      if (length > this.allowMovementAmount) return false
      if (targetStack.length === 0) return true
      const targetStackTopCard = this.cardType.get(targetStack[targetStack.length - 1].value)
      let draggingCardColor = this.getCardColor(card.suit)
      let targetTopCardColor = this.getCardColor(targetStackTopCard.suit)
      if (draggingCardColor === targetTopCardColor) {
        return false
      } else {
        if ((card.number + 1) === targetStackTopCard.number) {
          return true
        } else {
          return false
        }
      }
    },
    calculateAllStacksDragable () {
      for (let i = 0; i < this.cards.length; i++) {
        this.cardInStackIsDraggable(this.cards[i])
      }
    },
    freeSpaceCanDrop (targetId = -1) {
      if (targetId === -1) {
        return (!!~this.freeSpace.indexOf(0))
      } else {
        return this.freeSpace[targetId] === 0
      }
    },
    /**
     * @param {PokerBasic} card
     */
    targetSpaceCanDrop (card, targetId = -1) {
      if (card.number === 1) return true
      if (targetId === -1) {
        if (~(this.targetSpace.findIndex(e => this.cardType.get(e + 1) === card))) {
          return true
        }
      } else {
        if (this.cardType.get(this.targetSpace[targetId] + 1) === card) {
          return true
        }
      }
      return false
    },
    /**
     * @param {number} freeSpaceId
     */
    tmpMoveToFreeSpace (freeSpaceId) {
      this.clicked = -1
      if (this.tmpDragList.from === 'freeSpace') {
        if (this.tmpDragList.cardsPos.i === freeSpaceId) return true
        else if (this.freeSpace[freeSpaceId] === 0) {
          [this.freeSpace[freeSpaceId], this.freeSpace[this.tmpDragList.cardsPos.i]] = [this.freeSpace[this.tmpDragList.cardsPos.i], this.freeSpace[freeSpaceId]]
        }
      } else {
        let { i } = this.tmpDragList.cardsPos
        this.freeSpace[freeSpaceId] = this.cards[i].pop()
      }
      this.pushToMemoryStack(this.tmpDragList.from, 'freeSpace', this.tmpDragList.cardsPos, { i: freeSpaceId })
      this.tmpDragList = false
    },
    /**
     * @param {number} stackId
     */
    tmpMoveToStack (stackId) {
      this.clicked = -1
      if (this.tmpDragList.from === 'freeSpace') {
        // confirm the input card value is card stack last card value - 1
        this.cards[stackId].push(this.freeSpace[this.tmpDragList.cardsPos.i])
        this.freeSpace[this.tmpDragList.cardsPos.i] = 0
      } else {
        [].push.call(this.cards[stackId], ...this.cards[this.tmpDragList.cardsPos.i].splice(this.tmpDragList.cardsPos.j))
      }
      this.pushToMemoryStack(this.tmpDragList.from, 'stack', this.tmpDragList.cardsPos, { i: stackId })
      this.tmpDragList = false
    },
    /** @param {number} targetId */
    tmpMoveToTargetSpace (targetId = -1) {
      this.clicked = -1
      let card = this.tmpDragList.from === 'stack' ? this.cards[this.tmpDragList.cardsPos.i].pop() : this.freeSpace[this.tmpDragList.cardsPos.i]
      if (targetId === -1) {
        if (card.value % 13 === 1) {
          targetId = this.targetSpace.indexOf(0)
          this.targetSpace[targetId] = card.value
        } else {
          this.targetSpace.some((ele, i) => {
            if (this.cardType.get(ele + 1) === this.cardType.get(card.value)) {
              this.targetSpace[(targetId = i)] = card.value
              return true
            } else return false
          })
        }
      } else {
        this.targetSpace[targetId] = card.value
      }
      if (this.tmpDragList.from === 'freeSpace') {
        this.freeSpace[this.tmpDragList.cardsPos.i] = 0
      }
      this.pushToMemoryStack(this.tmpDragList.from, 'target', this.tmpDragList.cardsPos, { i: targetId })
      this.tmpDragList = false
      this.$forceUpdate()
    },
    forceMoveToTargetSpace (isRecursive = false) {
      let targetExpect = this.targetSpace.map(ele => {
        if (ele === 0) return '*'
        if (ele % 13 === 0) return 'finish'
        return this.cardType.get(ele + 1)
      })
      for (let i = 0; i < this.freeSpace.length; i++) {
        if (this.freeSpace[i] === 0) continue
        if (this.freeSpace[i].value % 13 === 1) {
          this.tmpDragList = {
            from: 'freeSpace',
            cardsPos: { i }
          }
          let targetIndex = targetExpect.indexOf('*')
          this.tmpMoveToTargetSpace(targetIndex)
          isRecursive = false
          targetExpect[targetIndex] = this.cardType.get(this.targetSpace[targetIndex])
        } else {
          let { suit, number } = this.cardType.get(this.freeSpace[i].value)
          let targetIndex = targetExpect.findIndex(ele => ele !== '*' && (ele.number === number && suit === ele.suit))
          if (!~targetIndex) continue
          this.tmpDragList = {
            from: 'freeSpace',
            cardsPos: { i }
          }
          this.tmpMoveToTargetSpace(targetIndex)
          isRecursive = false
          targetExpect[targetIndex] = JSON.parse(JSON.stringify({ suit, number: number + 1 > 13 ? 'finish' : number }))
        }
      }
      for (let i in this.cards) {
        if (this.cards[i].length === 0) continue
        let lastCard = this.cardType.get(_.last(this.cards[i]).value)
        if (lastCard.number === 1) {
          this.tmpDragList = {
            from: 'stack',
            cardsPos: {
              i,
              j: this.cards[i].length - 1,
              length: 1
            }
          }
          let targetIndex = targetExpect.indexOf('*')
          this.tmpMoveToTargetSpace(targetIndex)
          isRecursive = false
          targetExpect[targetIndex] = this.cardType.get(this.targetSpace[targetIndex])
        } else {
          this.tmpDragList = {
            from: 'stack',
            cardsPos: {
              i,
              j: this.cards[i].length - 1,
              length: 1
            }
          }
          let targetIndex = targetExpect.findIndex(ele => ele !== '*' && lastCard.number === ele.number && lastCard.suit === ele.suit)
          if (!~targetIndex) continue
          this.tmpMoveToTargetSpace(targetIndex)
          isRecursive = false
          let { suit, number } = lastCard
          targetExpect[targetIndex] = JSON.parse(JSON.stringify({ suit, number: number + 1 > 13 ? 'finish' : number }))
        }
      }
      if (isRecursive !== true) this.forceMoveToTargetSpace(true)
      else this.winOrLose()
    },
    scanAndMoveToTargetSpace (isRecursive = false) {
      if (this.autoScanNeededCardMem.get('*') < 4) {
        for (let i in this.cards) {
          if (this.cards[i].length === 0) continue
          let lastCard = _.last(this.cards[i])
          if (lastCard.value % 13 === 1 || (lastCard.value % 13 === 2 && this.targetSpaceCanDrop(this.cardType.get(lastCard.value)))) {
            this.tmpDragList = {
              from: 'stack',
              cardsPos: {
                length: 1,
                i: i,
                j: this.cards[i].length - 1
              }
            }
            if (lastCard.value % 13 === 1) {
              this.autoScanNeededCardMem.set('*', this.autoScanNeededCardMem.get('*') + 1)
            }
            this.tmpMoveToTargetSpace()
            this.tmpDragList = false
            isRecursive = false
          }
        }
        for (let i in this.freeSpace) {
          if (this.freeSpace[i].value % 13 === 1) {
            this.tmpDragList = { from: 'freeSpace', cardsPos: { i } }
            this.tmpMoveToTargetSpace()
            this.autoScanNeededCardMem.set('*', this.autoScanNeededCardMem.get('*') + 1)
            this.tmpDragList = false
            isRecursive = false
          }
        }
        if (!isRecursive && this.autoScanNeededCardMem.get('*') <= 4) {
          this.scanAndMoveToTargetSpace(true)
        }
      } else {
        let min = Math.min(...this.targetSpace.map(e => this.cardType.get(e).number))
        if (this.targetSpace.every(e => e === min)) min += 1
        for (let i in this.cards) {
          if (this.cards[i].length === 0) continue
          let card = this.cardType.get(_.last(this.cards[i]).value)
          if (card.number !== min + 1) continue
          let state = this.targetSpaceCanDrop(card)
          if (!state) continue
          this.tmpDragList = {
            from: 'stack', cardsPos: { length: 1, i, j: this.cards[i].length - 1 }
          }
          this.tmpMoveToTargetSpace()
          isRecursive = false
          this.tmpDragList = false
        }
        if (!this.freeSpace.every(e => e === 0)) {
          for (let i in this.freeSpace) {
            if (this.freeSpace[i] === 0) continue
            let card = this.cardType.get(this.freeSpace[i].value)
            if (card.number !== min + 1) continue
            let state = this.targetSpaceCanDrop(card)
            if (!state) continue
            this.tmpDragList = {
              from: 'freeSpace', cardsPos: { i }
            }
            this.tmpMoveToTargetSpace()
            this.tmpDragList = false
            isRecursive = false
          }
        }
        min = Math.min(...this.targetSpace.map(e => this.cardType.get(e).number))
        if (this.targetSpace.every(e => (((e - 1) % 13) + 1) === min) && !isRecursive) this.scanAndMoveToTargetSpace(true)
      }
    },
    /**
     * @desc {{from: 'stack', to: 'freeSpace', fromThe: {i: number, j:0}, toThe:{i:number}}} fromStackToFreeSpace
     * @desc {{from: 'stack', to: 'stack', fromThe: {i: number, j:number, length: number}, toThe:{i:number}}} fromStackToStack
     * @desc {{from: 'stack', to: 'target', fromThe: {i: number, j: number}}} fromStackToTarget
     * @desc {{from: 'freeSpace', to: 'stack', fromThe: {i:number}, toThe: {i:number}}} fromFreeSpaceToStack
     * @desc {{from: 'freeSpace', to: 'target', fromThe: {i:number}}} fromFreeSpaceToTarget
     * @returns {{from: string, to: string, fromThe: {i:number, j:number, length: number}, toThe:{i:number}}}
     *
     * @param {PokerBasic | false} tmpCard
     */
    hintingValidate (tmpCard = false) {
      let freeSpaceId = this.freeSpace.findIndex(e => e === 0)
      let stackNextPredict = new Map()
      let stackNextPredictIndex = new Set()
      this.cards.forEach((stack, i) => {
        if (stack.length === 0 ||
          _.last(stack).value % 13 === 1) {
        } else {
          let stackLast = _.last(stack)
          let card = this.cardType.get(stackLast.value)
          stackNextPredict.set(`${this.getCardColor(card.suit) === 'black' ? 'red' : 'black'}/${card.number - 1}`, i)
          stackNextPredictIndex.add((card.number - 1) % 13)
        }
      })
      for (let i = 0; i < this.cards.length; i++) {
        let stack = this.cards[i]
        if (stack.length === 0) continue
        if (stack.length === 1 && freeSpaceId !== -1) return { from: 'stack', to: 'freeSpace', fromThe: { i, j: 0 }, toThe: { i: freeSpaceId } }
        if (this.targetSpaceCanDrop(this.cardType.get(_.last(stack).value))) return { from: 'stack', to: 'target', fromThe: { i, j: stack.length - 1 } }
        let predictIndex = _.findLastIndex(stack, card => stackNextPredictIndex.has(card.value % 13))
        if (~predictIndex && !stack[predictIndex].draggable) continue
        let { number, suit } = this.cardType.get(stack[predictIndex].value)
        if (~predictIndex && stackNextPredict.has(`${this.getCardColor(suit)}/${number}`) && (stack.length - predictIndex < this.allowMovementAmount)) {
          return { from: 'stack', to: 'stack', fromThe: { i, j: predictIndex, length: stack.length - predictIndex }, toThe: { i: stackNextPredict.get(`${this.getCardColor(suit)}/${number}`) } }
        }
      }
      for (let i = 0; i < this.freeSpace.length; i++) {
        if (this.freeSpace[i] === 0) continue
        if (this.targetSpaceCanDrop(this.cardType.get(this.freeSpace[i].value))) {
          return { from: 'freeSpace', to: 'target', fromThe: { i } }
        }
        let card = this.cardType.get(this.freeSpace[i].value)
        if (stackNextPredictIndex.has(this.freeSpace[i].value % 13) &&
          stackNextPredict.has(`${this.getCardColor(card.suit)}/${card.number}`)) {
          return {
            from: 'freeSpace',
            to: 'stack',
            fromThe: {
              i
            },
            toThe: {
              i: stackNextPredict.get(`${this.getCardColor(card.suit)}/${card.number}`)
            }
          }
        }
      }
      return false
    },
    hintingOnClick () {
      this.hintingMap.clear()
      let result = this.hintingValidate()
      console.log(result)
      switch (result.from) {
        case 'stack':
          this.hintingMap.set(this.cards[result.fromThe.i][result.fromThe.j].value, 'from')
          break
        case 'freeSpace':
          this.hintingMap.set(this.freeSpace[result.fromThe.i].value, 'from')
          break
      }
    },
    winOrLose () {
      if (_.sum(this.targetSpace) === 130) {
        this.isWin = true
      } else if (this.allowMovementAmount !== 1 || this.hintingValidate()) {
        return false
      } else {
        this.isWin = false
      }
      clearInterval(this.timer)
      this.panelIsShow = true
    },
    pushToMemoryStack (from, to, fromThe, toThe) {
      this.memoryStack.push({ from, to, fromThe: JSON.parse(JSON.stringify(fromThe)), toThe: JSON.parse(JSON.stringify(toThe)) })
    },
    recoveryFromMemoryStack () {
      if (!this.memoryStack.length) return
      /**
       * @type {{from: 'freeSpace'|'stack', to: 'freeSpace'|'target'|'stack', fromThe: {i:number, j:number, length:number}, toThe: {i: number}}}
       */
      let mem = this.memoryStack.pop()
      if (mem.from === 'stack') {
        switch (mem.to) {
          case 'freeSpace':
            this.cards[mem.fromThe.i].push(this.freeSpace[mem.toThe.i])
            this.freeSpace[mem.toThe.i] = 0
            break
          case 'stack':
            this.cards[mem.fromThe.i].splice(this.cards[mem.fromThe.i].length, 0, ...this.cards[mem.toThe.i].splice(-mem.fromThe.length))
            break
          case 'target':
            this.cards[mem.fromThe.i].push({ value: this.targetSpace[mem.toThe.i], draggable: true })
            this.targetSpace[mem.toThe.i] = (this.targetSpace[mem.toThe.i] % 13) === 1 ? 0 : this.targetSpace[mem.toThe.i] - 1
            break
        }
        this.cardInStackIsDraggable(this.cards[mem.fromThe.i])
      } else {
        switch (mem.to) {
          case 'freeSpace':
            [this.freeSpace[mem.fromThe.i], this.freeSpace[mem.toThe.i]] = [this.freeSpace[mem.toThe.i], this.freeSpace[mem.fromThe.i]]
            break
          case 'stack':
            this.freeSpace[mem.fromThe.i] = this.cards[mem.fromThe.i].pop()
            break
          case 'target':
            this.freeSpace[mem.fromThe.i] = { value: this.targetSpace[mem.toThe.i], draggable: true }
            this.targetSpace[mem.toThe.i] = (this.targetSpace[mem.toThe.i] % 13) === 1 ? 0 : this.targetSpace[mem.toThe.i] - 1
            break
        }
      }
    },
    /**
     * @param {DragEvent} ev
     * @param {{i:number, j:number?}} cardPos
     * @param {string} from
     * @param {string} cardRef
     */
    cardOnDragStart (ev, cardPos, from, cardRef) {
      this.hintingMap.clear()
      this.$refs['ghost'].innerHTML = ''
      let cNode = this.$refs[cardRef][0].cloneNode(true)
      cNode.style.opacity = '1'
      cNode.style.marginBottom = '-160px'
      this.$refs['ghost'].appendChild(cNode)
      console.dir(ev.target)
      if (from === 'stack' && ev.target.parentNode.nextSibling) {
        let _target = ev.target.parentNode.nextSibling
        do {
          let _node = _target.firstChild.cloneNode(true)
          _node.style.marginBottom = '-160px'
          this.$refs['ghost'].appendChild(_target.cloneNode(true))
        } while ((_target = _target.nextElementSibling))
      }
      this.$refs['ghost'].style.opacity = '0.9'
      this.$refs['ghost'].style.paddingBottom = 160 * this.$refs['ghost'].childElementCount + 'px'
      ev.dataTransfer.setDragImage(this.$refs['ghost'], ev.offsetX, ev.offsetY)
      ev.dataTransfer.setData('text/plain', JSON.stringify({ cardPos, from }))
      return false
    },
    cardOnDragEnd () {
      this.scanAndMoveToTargetSpace()
      this.winOrLose()
      this.$refs['ghost'].innerHTML = ''
    },
    stackCardOnDBClick (i, j) {
      this.hintingMap.clear()
      this.tmpDragList = {
        from: 'stack',
        cardsPos: {
          i,
          j,
          length: 1
        }
      }
      let index = this.freeSpace.findIndex(s => s === 0)
      if (~index) {
        this.tmpMoveToFreeSpace(index)
      } else if (this.cards[i][j] && this.targetSpaceCanDrop(this.cardType.get(this.cards[i][j].value))) {
        this.tmpMoveToTargetSpace()
      }
      this.scanAndMoveToTargetSpace()
      this.cardInStackIsDraggable(this.cards[i])
      this.winOrLose()
    },
    /** @param {DragEvent} ev */
    freeSpaceOnDrop (ev, freeSpaceId = 0) {
      console.log('onDrop')
      let { cardPos, from } = JSON.parse(ev.dataTransfer.getData('text'))
      console.log(cardPos, from)
      if (from === 'stack') {
        if (this.$refs['ghost'].childElementCount === 1) {
          if (this.freeSpaceCanDrop(freeSpaceId)) {
            this.tmpDragList = {
              from: 'stack',
              cardsPos: {
                i: cardPos.i,
                j: cardPos.j,
                length: 1
              }
            }
            this.tmpMoveToFreeSpace(freeSpaceId)
          }
        }
      } else {
        if (this.freeSpaceCanDrop(freeSpaceId)) {
          this.tmpDragList = {
            from: 'freeSpace',
            cardsPos: {
              i: cardPos.i
            }
          }
          this.tmpMoveToFreeSpace(freeSpaceId)
        }
      }
    },
    /**
     * @param {DragEvent} ev
     * @param {number} targetId
     */
    targetOnDrop (ev, targetId) {
      let { cardPos, from } = JSON.parse(ev.dataTransfer.getData('text'))
      console.log(targetId)
      if (from === 'stack') {
        if (this.$refs['ghost'].childElementCount === 1) {
          let card = this.cardType.get(this.cards[cardPos.i][cardPos.j].value)
          console.log(card)
          if (this.targetSpaceCanDrop(card)) {
            this.tmpDragList = {
              from: 'stack',
              cardsPos: {
                length: 1,
                i: cardPos.i,
                j: cardPos.j
              }
            }
            this.tmpMoveToTargetSpace(targetId)
          }
        }
      } else {
        let freeSpaceCard = this.freeSpace[cardPos.i]
        let _card = this.cardType.get(freeSpaceCard.value)
        if (this.targetSpaceCanDrop(_card, targetId)) {
          this.tmpDragList = {
            from: 'freeSpace',
            cardsPos: {
              i: cardPos.i
            }
          }
          this.tmpMoveToTargetSpace(targetId)
        }
      }
      this.$refs['ghost'].innserHTML = ''
    },
    /**
     * @param {DragEvent} ev
     */
    stackOnDrop (ev, stackId) {
      let { cardPos, from } = JSON.parse(ev.dataTransfer.getData('text'))
      console.log(cardPos)
      if (from === 'stack') {
        if (this.$refs['ghost'].childElementCount > this.allowMovementAmount) {
          alert('No more card space movement, current is' + this.allowMovementAmount)
          return false
        }
        let targetCardStack = this.cards[stackId]
        let candidates = _.takeRight(this.cards[cardPos.i], this.$refs['ghost'].childElementCount)
        console.log(candidates, this.cards[cardPos.i])
        let firstCard = this.cardType.get(candidates[0].value)
        let canDrop = this.stackCanDrop(firstCard, targetCardStack)
        if (canDrop) {
          this.tmpDragList = {
            from: 'stack',
            cardsPos: {
              i: cardPos.i,
              j: cardPos.j,
              length: this.$refs['ghost'].childElementCount
            }
          }

          this.tmpMoveToStack(stackId)
        }
        console.log(canDrop)
      } else {
        let card = this.cardType.get(this.freeSpace[cardPos.i].value)
        let canDrop = this.stackCanDrop(card, this.cards[stackId])
        if (canDrop) {
          this.tmpDragList = {
            from: 'freeSpace',
            cardsPos: {
              i: cardPos.i,
              j: cardPos.j
            }
          }
          this.tmpMoveToStack(stackId)
        }
      }
      this.cardInStackIsDraggable(this.cards[stackId])
      // this.stackCanDrop()
    },
    getCard (num) {
      return card(`./${this.cardType.get(num).suit}_${this.cardType.get(num).number}.svg`)
    },
    cardOnClick (from, { i, j }) {
      if (this.tmpDragList) {
        if (this.tmpDragList.from === from && this.tmpDragList.cardsPos.i === i) {
          this.clicked = -1
          this.tmpDragList = false
          return
        }
        switch (from) {
          case 'freeSpace':
            if (this.freeSpaceCanDrop(i)) {
              this.tmpMoveToFreeSpace(i)
            }
            break
          case 'target':
            if (this.tmpDragList.from === 'stack') {
              if (!this.targetSpaceCanDrop(this.cardType.get(this.cards[this.tmpDragList.cardsPos.i].slice(-1)[0].value))) {
                this.tmpDragList = false
                return false
              }
            } else if (!this.targetSpaceCanDrop(this.cardType.get(this.freeSpace[this.tmpDragList.cardsPos.i]))) {
              this.tmpDragList = false
              return false
            }
            this.tmpMoveToTarget(i)
            break
          case 'stack':
            if (this.tmpDragList.from === 'stack') {
              let { number, suit } = this.cardType.get(_.last(this.cards[i]).value)
              if (number === 1) {
                this.scanAndMoveToTargetSpace()
                this.tmpDragList = false
                return false
              }
              let expected = { number: number - 1, suit: this.getCardColor(suit) === 'black' ? 'red' : 'black' }
              for (let i = this.tmpDragList.cardsPos.j; i < this.cards[this.tmpDragList.cardsPos.i].length; i++) {
                let { number, suit } = this.cardType.get(this.cards[this.tmpDragList.cardsPos.i][i].value)
                if (number === expected.number && this.getCardColor(suit) === expected.suit) {
                  if (i === (+this.tmpDragList.cardsPos.j) + i) {
                    break
                  } else {
                    this.tmpDragList.cardsPos.j = i
                    this.tmpDragList.cardsPos.length = this.cards[this.tmpDragList.cardsPos.i].length - i + 1
                    break
                  }
                }
              }
              if (!this.stackCanDrop(this.cardType.get(this.cards[this.tmpDragList.cardsPos.i][this.tmpDragList.cardsPos.j].value), this.cards[i], this.tmpDragList.cardsPos.length)) {
                alert('Invalid movement')
                this.tmpDragList = false
                return false
              }
            } else if (!this.stackCanDrop(this.cardType.get(this.freeSpace[this.tmpDragList.cardsPos.i].value), this.cards[i], 1)) {
              this.tmpDragList = false
              return false
            }
            this.tmpMoveToStack(i)
            break
        }
        this.scanAndMoveToTargetSpace()
        this.hintingMap.clear()
        if (from === 'stack' || this.tmpDragList.from === 'stack') {
          this.cardInStackIsDraggable(this.cards[i])
        }
      } else {
        switch (from) {
          case 'freeSpace':
            this.tmpDragList = {
              from,
              cardsPos: {
                i
              }
            }
            this.clicked = this.freeSpace[i].value
            console.log(this.clicked)
            break
          case 'stack':
            this.tmpDragList = {
              from,
              cardsPos: {
                i,
                j,
                length: this.cards[i].length - j
              }
            }
            this.clicked = _.last(this.cards[i]).value
            console.log(this.clicked)
            this.cardInStackIsDraggable(this.cards[i])
            break
        }
      }
    },
    panelOnClick (action) {
      switch (action) {
        case 'again':
          this.init()
          break
        case 'close':
          this.panelIsShow = false
          break
        case 'restart':
          this.init(true)
          break
        case 'renew':
          this.init()
          this.panelIsShow = false
          break
        case 'undo':
          this.recoveryFromMemoryStack()
          break
      }
    }
  }
}
</script>
<style lang="stylus">
size($w, $h = $w)
  height ($h / 3 )* 2
  width ($w / 3) * 2
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
    top (52px / 3) * 2
    left (180px / 3) * 2
    position absolute
  .round, .time, .score
    color #E6BF8D
    white-space nowrap
  .round
    size 129px 37px
    font-size (28px / 3) * 2
    position absolute
    left (597px / 3) * 2
    top (66px / 3) * 2
  .time
    position absolute
    size 218px 53px
    left (1292px / 3) * 2
    top (50px / 3) * 2
    font-size (40px / 3) * 2
    font-weight bold
  .score
    position absolute
    size 160px 53px
    left (1580px / 3) * 2
    top (50px / 3) * 2
    font-size (40px / 3) * 2
    font-weight bold
  .undo, .hint
    position absolute
    size (50px / 3) * 2
    cursor pointer
    right (66px / 3) * 2
  .group
    cursor pointer
    size 52px 43px
    position absolute
    right (64px / 3) * 2
    top (327px / 3) * 2
  .undo, .hint, .group
    transition 0.3s
    &:hover
      filter drop-shadow(0 0 5px white)
    &:active
      filter drop-shadow(0 0 2px white)
  .undo
    top (143px / 3) * 2
  .hint
    top (235px / 3) * 2
  .free-space
    size 706px 215px
    position absolute
    top (143px / 3) * 2
    left (180px / 3) * 2
    display flex
    justify-content space-between
    align-items center
    .free-space-ctn
      size 154px 215px
      border-radius (18px / 3) * 2
      display flex
      justify-content center
      align-items center
      background url('assets/hint_up/Component 2 – 32.svg')
      background-size cover
      position relative
  .target-space
    size 706px 215px
    position absolute
    top (143px / 3) * 2
    left (1034px / 3) * 2
    display flex
    justify-content space-between
    align-items center
    .target-ctn
      display flex
      justify-content center
      align-items center
      size 154px 215px
      border-radius (18px / 3) * 2
      position relative
      border 7px solid #FCCD8D
      background-color #171D29
  .bottom-space
    position absolute
    top (421px / 3) * 2
    left 50%
    transform translateX(-50%)
    display flex
    width (1508px / 3) * 2
    justify-content space-between
    align-content flex-start
    margin 0 auto
    .stack
      width (152px / 3) * 2
      min-height (215px / 3) * 2
      padding-bottom (160px / 3) * 2
      display inline-block
      &.tight .card
        margin-bottom (-170px / 3) * 2
      .card
        margin-bottom (-160px / 3) * 2
        position relative
        &[draggable='true']:hover
          img
            transform translateY(-15px)
            &:active
              transform translateY(-10px)
  img[draggable='true']
    cursor pointer
  img
    user-select none
  .card, .free-space-ctn, .target-ctn, .ghost
    img
      size 152px 215px
  img.is-click
    transform translateY(-10px)
    &:after
      transform translateY(-10px)
  .card.from:after, .free-space-ctn.from:after
    content ''
    position absolute
    left (-16px / 3) * 2
    top (-16px / 3) * 2
    size 184px 245px
    background url('./assets/hint_up/hint_up@2x.png')
    background-size contain
  .stack .card.from
    &:hover:after
      transform translateY(-15px)
    &:active:after
      transform translateY(-10px)
.ghost-space
  width 0
  height 0
  position absolute
  overflow hidden
.ghost
  width (152px / 3) * 2
  .card
    margin-bottom (-160px / 3) * 2
</style>
