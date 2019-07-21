<template>
  <div class="background"
    @click.prevent="$emit('input', false)">
    <div class="panel"
      v-if="isWin">
      <h1 class="win">WIN</h1>
      <div class="again">Play again?</div>
      <div class="btn-group">
        <div class="btn" @click="$emit('panel', 'again')">YES</div>
        <div class="btn" @click="$emit('panel', 'close')">NO</div>
      </div>
    </div>
    <div class="panel"
      @click.stop
      v-else>
      <h1 class="nomore">NO MORE MOVES</h1>
      <div class="question" @click="$emit('panel', 'restart')">Restart this Game</div>
      <div class="question" @click="$emit('panel', 'renew')">Quit and Start a New Game</div>
      <div class="question" @click="$emit('panel', 'undo')">Go back and undo some moves</div>
    </div>
  </div>
</template>
<script>
export default {
  props: ['isWin', 'value']
}
</script>

<style lang="stylus" scoped>
.background
  position fixed
  left 0
  top 0
  width 100%
  height 100%
  background-color rgba(black, 0.75)
.panel
  position absolute
  display flex
  flex-direction column
  align-items center
  box-sizing border-box
  top 235px
  left 50%
  transform translateX(-50%)
  width 912px
  height 401px
  text-align center
  background black
  box-shadow 0 3px 11px rgba(#000, 0.72)
  h1.nomore
    color #575757
    font-size 42px
    margin-top 49.71px
    margin-bottom 42.29px
  .question
    display flex
    justify-content center
    align-items center
    color black
    font-size 30px
    font-weight bold
    height 50px
    width 500px
    margin-bottom 27px
    background-color #C7A466
    border solid 2px #ECCC80
    border-radius 15px
    &:last-of-type
      margin-bottom 59px
  h1.win
    margin 0
    width 209.37px
    height 71px
    font-size 72px
    margin-top 46px
    color #E6BF8D
  .again
    margin-top 12px
    color #E6BF8D
    font-size 44px
    height 58px
    width 232px
    line-height 58px
  .btn-group
    margin-top 38px
    width 310px
    height 130px
    display flex
    justify-content space-between
    align-items center
    .btn
      color #0F1620
      background-color #E6BF8D
      display flex
      justify-content center
      align-items center
      font-size 50px
      width 130px
      height 130px
      border-radius 15px
  .question, .btn
    cursor pointer
</style>
