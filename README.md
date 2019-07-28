# the-f2e-w2-freecell

前端時光屋week2 - freecell

採用[設計稿](https://xd.adobe.com/spec/68f4497b-792e-4c0d-5d04-a519e2981b7f-1ec5/grid)

活動用[網址](https://challenge.thef2e.com/user/323?schedule=2967#works-2967)

[預覽網址](https://tianyili.github.io/the-f2e-w2-freecell)


注意，drag/drop api相關設計僅限chrome上能正常顯示，firefox經過測試無法達成想要的效果...

<del>最佳遊玩解析度為1920 * 1080(也就是說要放大至全螢幕，RWD以後有空再說...)</del>
目前設定最佳是1280 * 720，動態調整還是等有空再說XD

遊玩方式:

拖曳/點選來進行移動卡牌，double click可以直接將卡牌送上左上暫時存放區

可以進行恢復的動作

右鍵可以強制將允許的卡牌送上解答區域

右邊由上往下為

1. 恢復

2. Hint

3. 新牌局

Have fun~

> 題外話：這次製作光是code reindex 就讓我的pc有點卡了zzZZ, 後續應該不會再用那麼麻煩的算法了吧((X

## Project setup
```
yarn install
```

### Compiles and hot-reloads for development
```
yarn run serve
```

### Compiles and minifies for production
```
yarn run build
```

### Run your tests
```
yarn run test
```

### Lints and fixes files
```
yarn run lint
```

### Customize configuration
See [Configuration Reference](https://cli.vuejs.org/config/).
