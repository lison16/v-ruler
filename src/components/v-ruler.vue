<template>
  <div :style="wrapperStyle" class="vue-ruler-wrapper" onselectstart="return false;">
    <section v-show="rulerToggle">
      <div ref="levelRuler" class="vue-ruler-h" @mousedown.stop="levelDragRuler">
        <span v-for="(item,index) in xScale" :key="index" :style="{left:index * 50 + 2 + 'px'}" class="n">{{ item.id }}</span>
      </div>
      <div ref="verticalRuler" class="vue-ruler-v" @mousedown.stop="verticalDragRuler">
        <span v-for="(item,index) in yScale" :key="index" :style="{top:index * 50 + 2 + 'px'}" class="n">{{ item.id }}</span>
      </div>
      <div :style="{top:verticalDottedTop + 'px'}" class="vue-ruler-ref-dot-h" />
      <div :style="{left:levelDottedLeft + 'px'}" class="vue-ruler-ref-dot-v" />
      <div
        v-for="item in levelLineList"
        :title="item.title"
        :style="{top:item.top+ 'px'}"
        :key="item.id"
        class="vue-ruler-ref-line-h"
        @mousedown="dragLevelLine(item.id)"></div>
      <div
        v-for="item in verticalLineList"
        :title="item.title"
        :style="{left:item.left+ 'px'}"
        :key="item.id"
        class="vue-ruler-ref-line-v"
        @mousedown="dragVerticalLine(item.id)"></div>
    </section>
    <div ref="content" class="vue-ruler-content" :style="contentStyle">
      <slot />
    </div>
    <div v-show="isDrag" class="vue-ruler-content-mask"></div>
  </div>
</template>

<script>
import { on, off } from './event.js'
export default {
  name: 'VRuler',
  components: {},
  props: {
    position: {
      type: String,
      default: 'relative',
      validator: function (val) {
        return ['absolute', 'fixed', 'relative', 'static', 'inherit'].indexOf(val) !== -1
      }
    }, // 规定元素的定位类型
    isHotKey: {
      type: Boolean, default: true
    }, // 热键开关
    isScaleRevise: {
      type: Boolean, default: false
    }, // 刻度修正(根据content进行刻度重置)
    presetLine: {
      type: Array,
      default: () => {
        return [] // { type: 'l', site: 50 }, { type: 'v', site: 180 }
      }
    }, // 预置参考线
    contentLayout: {
      type: Object,
      default: () => {
        return { top: 0, left: 0 }
      }
    }, // 内容部分布局
    parent: {
      type: Boolean,
      default: false
    },
    visible: {
      type: Boolean,
      default: true
    }
  },
  data () {
    return {
      size: 17,
      left_top: 18, // 内容左上填充
      windowWidth: 0, // 窗口宽度
      windowHeight: 0, // 窗口高度
      xScale: [], // 水平刻度
      yScale: [], // 垂直刻度
      topSpacing: 0, // 标尺与窗口上间距
      leftSpacing: 0, //  标尺与窗口左间距
      isDrag: false,
      dragFlag: '', // 拖动开始标记，可能值x(从水平标尺开始拖动),y(从垂直标尺开始拖动)
      levelLineList: [], // 生成的水平线列表
      verticalLineList: [], // 生成的垂直线列表
      levelDottedLeft: -999, // 水平虚线位置
      verticalDottedTop: -999, // 垂直虚线位置
      rulerWidth: 0, // 垂直标尺的宽度
      rulerHeight: 0, // 水平标尺的高度
      dragLineId: '', // 被移动线的ID
      keyCode: {
        r: 82
      }, // 快捷键参数
      rulerToggle: true // 标尺辅助线显示开关
    }
  },
  computed: {
    wrapperStyle() {
      return {
        width : this.windowWidth + 'px',
        height : this.windowHeight + 'px',
        position: this.position
      }
    },
    contentStyle() {
      return {
        left: this.contentLayout.left + 'px',
        top: this.contentLayout.top + 'px',
        padding: this.left_top + 'px 0px 0px ' + this.left_top + 'px'
      }
    }
  },
  watch: {
    visible: {
      handler(visible) {
        this.rulerToggle = visible;
      },
      immediate: true
    }
  },
  mounted () {
    on(document, 'mousemove', this.dottedLineMove)
    on(document, 'mouseup', this.dottedLineUp)
    on(document, 'keyup', this.keyboard)
    this.init()
    this.quickGeneration(this.presetLine) // 生成预置参考线
    const self = this // 绑定窗口调整大小onresize事件
    window.onresize = function () { // 如果直接使用this,this指向的不是vue实例
      self.xScale = []
      self.yScale = []
      self.init()
    }
  },
  beforeDestroy () {
    off(document, 'mousemove', this.dottedLineMove)
    off(document, 'mouseup', this.dottedLineUp)
    off(document, 'keyup', this.keyboard)
  },
  methods: {
    init () {
      this.box()
      this.scaleCalc()
    },
    box () {
      if (this.isScaleRevise) { // 根据内容部分进行刻度修正
        const content = this.$refs.content
        const contentLeft = content.offsetLeft
        const contentTop = content.offsetTop
        for (let i = 0; i < contentLeft; i += 1) {
          if (i % 50 === 0 && i + 50 <= contentLeft) {
            this.xScale.push({ id: i })
          }
        }
        for (let i = 0; i < contentTop; i += 1) {
          if (i % 50 === 0 && i + 50 <= contentTop) {
            this.yScale.push({ id: i })
          }
        }
      }
      if (this.parent) {
        const style = window.getComputedStyle(this.$el.parentNode, null)
        this.windowWidth = parseInt(style.getPropertyValue('width'), 10)
        this.windowHeight = parseInt(style.getPropertyValue('height'), 10)
      } else {
        this.windowWidth = document.documentElement.clientWidth - this.leftSpacing
        this.windowHeight = document.documentElement.clientHeight - this.topSpacing
      }
      this.rulerWidth = this.$refs.verticalRuler.clientWidth
      this.rulerHeight = this.$refs.levelRuler.clientHeight
      this.setSpacing()
    }, // 获取窗口宽与高
    setSpacing () {
      this.topSpacing = this.$refs.levelRuler.getBoundingClientRect().y //.offsetParent.offsetTop
      this.leftSpacing = this.$refs.verticalRuler.getBoundingClientRect().x// .offsetParent.offsetLeft
    },
    scaleCalc () {
      for (let i = 0; i < this.windowWidth; i += 1) {
        if (i % 50 === 0) {
          this.xScale.push({ id: i })
        }
      }
      for (let i = 0; i < this.windowHeight; i += 1) {
        if (i % 50 === 0) {
          this.yScale.push({ id: i })
        }
      }
    }, // 计算刻度
    newLevelLine () {
      this.isDrag = true
      this.dragFlag = 'x'
    }, // 生成一个水平参考线
    newVerticalLine () {
      this.isDrag = true
      this.dragFlag = 'y'
    }, // 生成一个垂直参考线
    dottedLineMove ($event) {
      this.setSpacing()
      switch (this.dragFlag) {
        case 'x':
          if (this.isDrag) {
            this.verticalDottedTop = $event.pageY - this.topSpacing
          }
          break
        case 'y':
          if (this.isDrag) {
            this.levelDottedLeft = $event.pageX - this.leftSpacing
          }
          break
        case 'l':
          if (this.isDrag) {
            this.verticalDottedTop = $event.pageY - this.topSpacing
          }
          break
        case 'v':
          if (this.isDrag) {
            this.levelDottedLeft = $event.pageX - this.leftSpacing
          }
          break
        default:
          break
      }
    }, // 虚线移动
    dottedLineUp ($event) {
      this.setSpacing()
      if (this.isDrag) {
        this.isDrag = false
        switch (this.dragFlag) {
          case 'x':
            this.levelLineList.push(
              {
                id: 'levelLine' + this.levelLineList.length + 1,
                title: $event.pageY - this.topSpacing - this.size + 'px',
                top: $event.pageY - this.topSpacing
              }
            )
            break
          case 'y':
            this.verticalLineList.push(
              {
                id: 'verticalLine' + this.verticalLineList.length + 1,
                title: $event.pageX - this.leftSpacing - this.size + 'px',
                left: $event.pageX - this.leftSpacing
              }
            )
            break
          case 'l':
            if ($event.pageY - this.topSpacing < this.rulerHeight) {
              let Index, id
              this.levelLineList.forEach((item, index) => {
                if (item.id === this.dragLineId) {
                  Index = index
                  id = item.id
                }
              })
              this.levelLineList.splice(Index, 1, {
                id: id,
                title: -600 + 'px',
                top: -600
              })
            } else {
              let Index, id
              this.levelLineList.forEach((item, index) => {
                if (item.id === this.dragLineId) {
                  Index = index
                  id = item.id
                }
              })
              this.levelLineList.splice(Index, 1, {
                id: id,
                title: $event.pageY - this.topSpacing - this.size + 'px',
                top: $event.pageY - this.topSpacing
              })
            }
            break
          case 'v':
            if ($event.pageX - this.leftSpacing < this.rulerWidth) {
              let Index, id
              this.verticalLineList.forEach((item, index) => {
                if (item.id === this.dragLineId) {
                  Index = index
                  id = item.id
                }
              })
              this.verticalLineList.splice(Index, 1, {
                id: id,
                title: -600 + 'px',
                left: -600
              })
            } else {
              let Index, id
              this.verticalLineList.forEach((item, index) => {
                if (item.id === this.dragLineId) {
                  Index = index
                  id = item.id
                }
              })
              this.verticalLineList.splice(Index, 1, {
                id: id,
                title: $event.pageX - this.leftSpacing - this.size + 'px',
                left: $event.pageX - this.leftSpacing
              })
            }
            break
          default:
            break
        }
        this.verticalDottedTop = this.levelDottedLeft = -10
      }
    }, // 虚线松开
    levelDragRuler () {
      this.newLevelLine()
    }, // 水平标尺处按下鼠标
    verticalDragRuler () {
      this.newVerticalLine()
    }, // 垂直标尺处按下鼠标
    dragLevelLine (id) {
      this.isDrag = true
      this.dragFlag = 'l'
      this.dragLineId = id
    }, // 水平线处按下鼠标
    dragVerticalLine (id) {
      this.isDrag = true
      this.dragFlag = 'v'
      this.dragLineId = id
    }, // 垂直线处按下鼠标
    keyboard ($event) {
      if (this.isHotKey) {
        switch ($event.keyCode) {
          case this.keyCode.r:
            this.rulerToggle = !this.rulerToggle
            this.$emit('update:visible', this.rulerToggle)
            if (this.rulerToggle) {
              this.left_top = 18;
            } else {
              this.left_top = 0;
            }
            break
        }
      }
    }, // 键盘事件
    quickGeneration (params) {
      if (params !== []) {
        params.forEach(item => {
          switch (item.type) {
            case 'l':
              this.levelLineList.push({
                id: 'levelLine' + this.levelLineList.length + 1,
                title: item.site + 'px',
                top: item.site + this.size
              })
              break
            case 'v':
              this.verticalLineList.push({
                id: 'verticalLine' + this.verticalLineList.length + 1,
                title: item.site + 'px',
                left: item.site + this.size
              })
              break
            default:
              break
          }
        })
      }
    } // 快速生成参考线
  }
}
</script>

<style lang="scss">
.vue-ruler{
  &-wrapper {
    left: 0;
    top: 0;
    z-index: 999;
    overflow: hidden;
    user-select: none;
  }
  &-h,
  &-v,
  &-ref-line-v,
  &-ref-line-h,
  &-ref-dot-h,
  &-ref-dot-v {
    position: absolute;
    left: 0;
    top: 0;
    overflow: hidden;
    z-index: 999;
  }
  &-h,
  &-v,
  &-ref-line-v,
  &-ref-line-h,
  &-ref-dot-h,
  &-ref-dot-v {
    position: absolute;
    left: 0;
    top: 0;
    overflow: hidden;
    z-index: 999;
  }

  &-h {
    width: 100%;
    height: 18px;
    left: 18px;
    opacity: 0.6;
    background: url(data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAASCAMAAAAuTX21AAAAGXRFWHRTb2Z0d2FyZQBBZG9iZSBJbWFnZVJlYWR5ccllPAAAAAlQTFRFMzMzAAAA////BqjYlAAAACNJREFUeNpiYCAdMDKRCka1jGoBA2JZZGshiaCXFpIBQIABAAplBkCmQpujAAAAAElFTkSuQmCC)
      repeat-x; /*./image/ruler_h.png*/
  }

  &-v {
    width: 18px;
    height: 100%;
    top: 18px;
    opacity: 0.6;
    background: url(data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAABIAAAAyCAMAAABmvHtTAAAAGXRFWHRTb2Z0d2FyZQBBZG9iZSBJbWFnZVJlYWR5ccllPAAAAAlQTFRFMzMzAAAA////BqjYlAAAACBJREFUeNpiYGBEBwwMTGiAakI0NX7U9aOuHyGuBwgwAH6bBkAR6jkzAAAAAElFTkSuQmCC)
      repeat-y; /*./image/ruler_v.png*/
  }

  &-v .n,
  &-h .n {
    position: absolute;
    font: 10px/1 Arial, sans-serif;
    color: #333;
    cursor: default;
  }

  &-v .n {
    width: 8px;
    left: 3px;
    word-wrap: break-word;
  }

  &-h .n {
    top: 1px;
  }

  &-ref-line-v,
  &-ref-line-h,
  &-ref-dot-h,
  &-ref-dot-v {
    z-index: 998;
  }

  &-ref-line-h {
    width: 100%;
    height: 3px;
    background: url(data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAgAAAABCAMAAADU3h9xAAAAGXRFWHRTb2Z0d2FyZQBBZG9iZSBJbWFnZVJlYWR5ccllPAAAAAZQTFRFSv//AAAAH8VRuAAAAA5JREFUeNpiYIACgAADAAAJAAE0lmO3AAAAAElFTkSuQmCC)
      repeat-x left center; /*./image/line_h.png*/
    cursor: n-resize; /*url(./image/cur_move_h.cur), move*/
  }

  &-ref-line-v {
    width: 3px;
    height: 100%;
    _height: 9999px;
    background: url(data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAAICAMAAAAPxGVzAAAAGXRFWHRTb2Z0d2FyZQBBZG9iZSBJbWFnZVJlYWR5ccllPAAAAAZQTFRFSv//AAAAH8VRuAAAAA5JREFUeNpiYEAFAAEGAAAQAAGePof9AAAAAElFTkSuQmCC)
      repeat-y center top; /*./image/line_v.png*/
    cursor: w-resize; /*url(./image/cur_move_v.cur), move*/
  }

  &-ref-dot-h {
    width: 100%;
    height: 3px;
    background: url(data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAIAAAACCAMAAABFaP0WAAAAGXRFWHRTb2Z0d2FyZQBBZG9iZSBJbWFnZVJlYWR5ccllPAAAAAZQTFRFf39/////F3PnHQAAAAJ0Uk5T/wDltzBKAAAAEElEQVR42mJgYGRgZAQIMAAADQAExkizYQAAAABJRU5ErkJggg==)
      repeat-x left 1px; /*./image/line_dot.png*/
    cursor: n-resize; /*url(./image/cur_move_h.cur), move*/
    top: -10px;
  }

  &-ref-dot-v {
    width: 3px;
    height: 100%;
    _height: 9999px;
    background: url(data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAIAAAACCAMAAABFaP0WAAAAGXRFWHRTb2Z0d2FyZQBBZG9iZSBJbWFnZVJlYWR5ccllPAAAAAZQTFRFf39/////F3PnHQAAAAJ0Uk5T/wDltzBKAAAAEElEQVR42mJgYGRgZAQIMAAADQAExkizYQAAAABJRU5ErkJggg==)
      repeat-y 1px top; /*./image/line_dot.png*/
    cursor: w-resize; /*url(./image/cur_move_v.cur), move*/
    left: -10px;
  }
  &-content {
    position: absolute;
    z-index: 997;
  }
  &-content-mask{
    position: absolute;
    width: 100%;
    height: 100%;
    background: transparent;
    z-index: 998;
  }
}




</style>
