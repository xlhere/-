# CSS总结



[toc]

## 1.CSS实现一个秒针效果（一分钟转一圈，匀速和一秒一走）

```
    .item {
      background: black;
      border-radius: 50%;
      width: 6px;
      height: 200px;
      position: absolute;
      left: 50%;
      margin-left: -3px;
      animation-name: rotateCircle;
      animation-duration: 60s;
      /* 一秒一转 */
      animation-timing-function: steps(60);
      /* 匀速 */
      /* animation-timing-function: linear; */
      /* 无限播放 */
      animation-iteration-count: infinite;
      /* 旋转基点设置为底部居中 */
      transform-origin: center bottom;
    }
    @keyframes rotateCircle {
      0% {
        transform: rotate(0deg);
      }

      100% {
        transform: rotate(360deg);
      }
    }
```

[demo](https://github.com/onechunlin/JS_Demo/tree/master/CSS实现秒针)

##2.CSS实现类似微信朋友圈的效果，要求根据图片数量显示不同的布局

```css
/* 倒数第二张图片也是第二张图片，即有两张图片时 */
li:nth-last-child(2):first-child,
/* 第一张图片之后的样式 */
li:nth-last-child(2):first-child ~li{
    width: calc(100% / 2);
    height: calc(100% / 2);
}
```

[demo](https://github.com/onechunlin/JS_Demo/tree/master/CSS实现不同内容数量不同布局)

## 3.position的取值

