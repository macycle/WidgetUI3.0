<template>
  <div class="wt-carousel" @touchstart="touchStart" @touchmove="touchMove" @touchend="touchEnd">
    <div
    v-if="options.direction == 'horizontal'"
    :class="options.direction + ' wrapper'"
    :style="{transform: 'translate3d('+ state.distance +'px, 0px, 0px)'}">
      <slot></slot>
    </div>
    <div
    v-else-if="options.direction == 'vertical'"
    :class="options.direction + ' wrapper'"
    :style="{transform: 'translate3d(0px, '+ state.distance +'px, 0px)', height:(options.height || state.itemWidth) + 'px'}">
      <slot></slot>
    </div>
    <div :class="options.direction + ' carousel-pagination'" v-if="options.pagination">
      <p v-for="(item, index) in state.itemCount" 
      :key="index" 
      :class="{'active':index === state.currentIndex}"
      :style="{background:index === state.currentIndex ? options.paginationColor : ''}"
      ></p>
    </div>
  </div>
</template>
<script>
import { reactive, onMounted, getCurrentInstance } from "vue";

export default {
  props: {
    options: {
      type: Object,
      default: () => {
        return {
          loop: false, // 循环
          pagination: true, // 指示器
          auto: false, // 自动
          time: 3000, // 播放间隔
          paginationColor: '#fff', // 当前指示器颜色
          direction: 'horizontal' // 水平方向 (vertical 垂直方向)
        };
      }
    },
    onSwiper:{
      type: Function,
      default:() =>{}
    }
  },
  setup(props, {emit}) {
    const { ctx } = getCurrentInstance();
    const state = reactive({
      itemWidth: 0, // item的宽度
      speed: 1, // 速度
      currentIndex: 0, // 当前处于第几个
      distance: 0, // 滑动距离
      itemCount: 0, // 有多少item
      thisDistanceX: 0, // 本次滑动距离X
      thisDistanceY: 0, // 本次滑动距离Y
      start: { X: 0, Y: 0 }, // 触摸坐标
      move: { X: 0, Y: 0 }, // 移动坐标
      slots: [], // 父组件的slot
      autoPlay: "", // 自动轮播定时器
      status: "" // 当前状态
    });
    // 定时器函数
    const autoPlayFn = () => {
      const firstChild = ctx.$el.firstChild;
      firstChild.style.transitionDuration = "300ms";
      // 循环轮播
      if (props.options.loop) {
        state.currentIndex++;
        // 当前索引大于itemCount ，就返回到第一个
        if (state.currentIndex > state.itemCount) {
          state.currentIndex = 0;
          firstChild.style.transitionDuration = "0ms";
          if (props.options.direction === 'horizontal') {
            state.distance = -state.itemWidth;
          } else if (props.options.direction === 'vertical') {
            state.distance = -props.options.height;
          }
        } else {
         if (props.options.direction === 'horizontal') {
           state.distance = -((state.currentIndex + 1) * state.itemWidth);
          } else if (props.options.direction === 'vertical') {
            state.distance = -((state.currentIndex + 1) * props.options.height);
          }
          
          if (state.currentIndex === state.itemCount) {
            state.currentIndex = 0;
            // 400ms之后滚动到第一张，并且过渡时间为0, 时间一定要设置大于300ms，否则会有闪屏问题
            setTimeout(() => {
              firstChild.style.transitionDuration = "0ms";
              if (props.options.direction === 'horizontal') {
               state.distance = -state.itemWidth;
              } else if (props.options.direction === 'vertical') {
                state.distance = -props.options.height;
              }
              
            }, 400);
          }
        }
      } else {
        // 不循环轮播
        if (state.currentIndex === state.itemCount - 1) {
          state.currentIndex = 0;
        } else {
          state.currentIndex++;
        }
         if (props.options.direction === 'horizontal') {
          state.distance = -(state.currentIndex * state.itemWidth);
          } else if (props.options.direction === 'vertical') {
           state.distance = -(state.currentIndex * props.options.height);
          }
      }
      emit("swiper", state.currentIndex);
    };
    onMounted(() => {
      state.itemWidth = ctx.$el.offsetWidth;
      const len = ctx.$el.children[0].children.length;
      // 处理每个item的宽度
      for (let i = 0; i < len; i++) {
        // if (ctx.$el.children[0].children[i].className.indexOf('carousel-item') > -1) {
          const elm = ctx.$el.children[0].children[i];
          elm.style.width = state.itemWidth + "px"; // 设置每个item的宽度
          state.slots.push(ctx.$el.children[0].children[i]);
          state.itemCount++;
          // 垂直方向，需要设置高度
          if (props.options.direction === 'vertical' && (props.options.height || state.itemWidth)) {
             elm.style.height = (props.options.height || state.itemWidth) + "px";
          }
        // }
      }
      // 需要重复轮播
      if (props.options.loop) {
        const firstChild = ctx.$el.firstChild;
        // 克隆第一个item
        const first = state.slots[0].cloneNode(true);
        firstChild.appendChild(first);
        // 克隆最后一个item
        const last = state.slots[state.slots.length - 1].cloneNode(true);
        firstChild.insertBefore(last, firstChild.childNodes[0]);
        firstChild.style.transitionDuration = "0ms";
        
        if (props.options.direction === 'horizontal') {
          state.distance = -state.itemWidth;
        } else if (props.options.direction === 'vertical') {
         state.distance = -props.options.height;
        }
      }
      // 是否自动轮播
      if (props.options.auto) {
        state.autoPlay = setInterval(autoPlayFn, props.options.time);
      }
    });
    const touchStart = event => {
      // 关闭定时器
      clearInterval(state.autoPlay);
      // 触摸坐标
      state.start.X = event.touches[0].clientX;
      state.start.Y = event.touches[0].clientY;
    };
    const touchMove = event => {
      state.status = "moveing";
      const firstChild = ctx.$el.firstChild;
      firstChild.style.transitionDuration = "0s";
      // 滑动时候的坐标
      state.move.X = event.touches[0].clientX;
      state.move.Y = event.touches[0].clientY;
      // 如果是横着滑动则阻止浏览器默认行为
      if (
        Math.abs(state.move.Y - state.start.Y) <
        Math.abs(state.move.X - state.start.X) || props.options.direction === 'vertical'
      ) {
        event.preventDefault();
        event.stopPropagation();
      } else if(props.options.direction === 'horizontal') {
        // 如果是竖着滑动，则 return
        return;
      }
      // 本次滑动距离X
      const thisDistanceX = state.move.X - state.start.X;
      state.thisDistanceX = thisDistanceX;
      // 本次滑动距离X
      const thisDistanceY = state.move.Y - state.start.Y;
      state.thisDistanceY = thisDistanceY;
      // 如果是循环轮播，则是currentIndex + 1
      let resistance = 1; // 滑动阻力
      if (
        !props.options.loop && ((thisDistanceX > 0 && state.currentIndex === 0) ||
        (thisDistanceX < 0 && state.currentIndex === state.itemCount - 1))
      ) {
        resistance = 0.2;
      }
      // distance 滑动距离
      let index = state.currentIndex;
      if (props.options.loop) {
        index = state.currentIndex + 1;
      }
      if (props.options.direction === 'horizontal') {
        state.distance = -(index * state.itemWidth) + thisDistanceX * resistance;
      } else if (props.options.direction === 'vertical') {
       state.distance = -(index * props.options.height) + thisDistanceY * resistance;
      }
    };
    // 是否循环
    const isLoop = () => {
      let index = state.currentIndex;
      if (props.options.loop) {
        index = state.currentIndex + 1;
      }
      if (props.options.direction === 'horizontal') {
        state.distance = -(state.itemWidth * index);
      } else if (props.options.direction === 'vertical') {
       state.distance = -(props.options.height * index);
      }
    };
    const touchEnd = () => {
      const firstChild = ctx.$el.firstChild;
      // 重新开启定时器
      if (props.options.auto) {
        state.autoPlay = setInterval(autoPlayFn, props.options.time);
      }
      // 如果是竖着滑动，则 return
      if (
        Math.abs(state.move.Y - state.start.Y) >
        Math.abs(state.move.X - state.start.X) &&
        props.options.direction === 'horizontal'
      ) {
        return;
      }
      // 如果state.status === ""说明没有移动
      if (state.status === "") {
        return;
      }
      firstChild.style.transitionDuration = "300ms";
      // 往左滑 || 往上滑
      if ((state.thisDistanceX < 0 && props.options.direction === 'horizontal') || (state.thisDistanceY < 0 && props.options.direction === 'vertical')) {
         let flag = false;
        if (state.thisDistanceX <= -(state.itemWidth / 3) || state.thisDistanceY <= -(props.options.height /3)) {
          // 当前index 必须小于item数量减1
          if (state.currentIndex < state.itemCount - 1) {
            state.currentIndex++;
          } else {
            if (props.options.loop && state.currentIndex < state.itemCount) {
              state.currentIndex++;
            }
          }
          flag = true;
        }
        isLoop();
        // 如果是无限循环，则需要把索引变成0
        let index = state.currentIndex;
        if (props.options.loop) {
          if (state.itemCount === state.currentIndex) {
            state.currentIndex = 0;
            index = 0;
            setTimeout(() => {
              firstChild.style.transitionDuration = "0s";
               if (props.options.direction === 'horizontal') {
                  state.distance = -state.itemWidth;
                } else if (props.options.direction === 'vertical') {
                 state.distance = -props.options.height;
                }
            }, 300);
          }
        }
        // 触发父级事件
        if(flag) {
          emit('swiper', index)
        }
      } else if((state.thisDistanceX > 0 && props.options.direction === 'horizontal') || (state.thisDistanceY > 0 && props.options.direction === 'vertical')){
        // 往右滑 || 往上滑
        let flag = false;
        if (state.thisDistanceX >= state.itemWidth / 3 || state.thisDistanceY >= props.options.height / 3) {
          
          // 当前index 必须小于item数量减1
          state.currentIndex--;
          if (!props.options.loop && state.currentIndex <= 0) {
            state.currentIndex = 0;
          }
          flag = true;
        }
        isLoop();
        // 如果是无限循环，则需要把索引变成最后一个
        let index = state.currentIndex;
        if (props.options.loop) {
          if (state.currentIndex < 0) {
            state.currentIndex = index = state.itemCount - 1;
            setTimeout(() => {
              firstChild.style.transitionDuration = "0s";
              if (props.options.direction === 'horizontal') {
                 state.distance = -(state.itemWidth * state.itemCount - 1);
              } else if (props.options.direction === 'vertical') {
                state.distance = -(props.options.height * state.itemCount - 1);
              }
            }, 300);
          }
        }
        // 触发父级事件
        if(flag) {
          emit('swiper', index)
        }
      }
      state.start.X = 0;
      state.move.X = 0;
      state.status = "";
    };
    return {
      state,
      touchStart,
      touchMove,
      touchEnd
    };
  }
};
</script>
<style lang="less" rel="stylesheet/less" scoped>
.wt-carousel {
  width: 100%;
  // height: 10rem;
  position: relative;
  overflow: hidden;
  .wrapper {
    width: 100%;
    height: 100%;
    display: -webkit-box;
    transition-duration: 300ms;
    transition-timing-function: ease-out;
    transition-property: transform;
    &.vertical {
      display: flow-root;
      flex-direction: column;
    }
    .carousel-item {
      text-align: center;
      font-size: 0.8rem;
      height: 100%;
      width: 100%;
      background: #fff;
    }
    img {
      height: 100%;
      width: 100%;
      float: left;
    }
  }
  .carousel-pagination {
    display: flex;
    justify-content: center;
    position: absolute;
    bottom: 0.3rem;
    width: 100%;
    &.vertical {
      right: 0;
      top: 0;
      width: 30px;
      flex-direction: column;
      justify-content: center;
    }
    p {
      margin: 0.2rem;
      height: 0.3rem;
      width: 0.3rem;
      border-radius: 0.3rem;
      background: #000;
      opacity: 0.4;
      &.active {
        background: #fff;
        opacity: 1;
      }
    }
  }
}
</style>
