<template>
    <div ref="item"
         class="vue-grid-item"
         :class="{
             'vue-resizable' : resizable,
             'resizing' : isResizing,
             'vue-draggable-dragging' : isDragging,
             'cssTransforms' : useCssTransforms,
             'disable-userselect': isDragging
         }"
         :style="style"
    >
        <!-- @todo add more named slots -->
        <slot></slot>
        <span v-if="resizable" ref="handle" :class="resizableHandleClass"></span>
    </div>
</template>
<script>
  import {
    setTopLeft,
    setTransform,
    createMarkup,
    getLayoutItem
  } from '../utils/utils.js'
  import {
    getControlPosition,
    offsetXYFromParentOf,
    createCoreData
  } from '../utils/draggableUtils.js'

  let interact = require('interactjs')

  export default {
    name: 'GridItem',
    props: {
      /**
       * Boolean flag for the facet to allow draggable handle.
       * Non-required; defaults to true.
       */
      isDraggable: {
        type: Boolean,
        required: false,
        default: true
      },
      /**
       * Boolean option for facet if it's resizable via bottom right handle.
       * Non-required; defaults to true.
       */
      isResizable: {
        type: Boolean,
        required: false,
        default: true
      },
      /**
       * Number that sets the Minimum Height. (@todo Find what this is.  pixels?  Row?)
       * Non-required; defaults to 1
       */
      minH: {
        type: Number,
        required: false,
        default: 1
      },
      /**
       * Number that sets the Minimum Width. (@todo Find what this is.  pixels?  Row?)
       * Non-required; defaults to 1
       */
      minW: {
        type: Number,
        required: false,
        default: 1
      },
      /**
       * Number that sets the Maximum Height. (@todo Find what this is.  pixels?  Row?)
       * Non-required; defaults to Infinity
       */
      maxH: {
        type: Number,
        required: false,
        default: Infinity
      },
      /**
       * Number that sets the Maximum Width. (@todo Find what this is.  pixels?  Row?)
       * Non-required; defaults to Infinity
       */
      maxW: {
        type: Number,
        required: false,
        default: Infinity
      },
      /**
       * Number that is the 'column' starting point of facet.
       * Required, no default.
       */
      x: {
        type: Number,
        required: true
      },
      /**
       * Number that is the 'row' starting point of facet.
       * Required, no default.
       */
      y: {
        type: Number,
        required: true
      },
      /**
       * Number that is the width of the facet; in 'columns'
       * Required, no default.
       */
      w: {
        type: Number,
        required: true
      },
      /**
       * Number that is the height of the facet; in 'row'
       * Required, no default.
       */
      h: {
        type: Number,
        required: true
      },
      /**
       * The content of the facet.
       * Required, no default
       */
      i: {
        required: true
      },
      /**
       * @todo FIgure out what this is.
       */
      dragIgnoreFrom: {
        type: String,
        required: false,
        default: 'a, button'
      },
      /**
       * @todo FIgure out what this is.
       */
      dragAllowFrom: {
        type: String,
        required: false,
        default: null
      },
      resizeIgnoreFrom: {
        type: String,
        required: false,
        default: 'a, button'
      }
    },
    inject: ['eventBus'],
    data: function () {
      return {
        cols: 1,
        containerWidth: 100,
        rowHeight: 30,
        margin: [10, 10],
        maxRows: Infinity,
        draggable: null,
        resizable: null,
        useCssTransforms: true,

        isDragging: false,
        dragging: null,
        isResizing: false,
        resizing: null,
        lastX: NaN,
        lastY: NaN,
        lastW: NaN,
        lastH: NaN,
        style: {},

        dragEventSet: false,
        resizeEventSet: false,

        previousW: null,
        previousH: null,
        previousX: null,
        previousY: null,
        innerX: this.x,
        innerY: this.y,
        innerW: this.w,
        innerH: this.h
      }
    },
    created () {
      var self = this;

      /**
       * Accessible references of functions for removing in beforeDestroy
       * @param {number} width - the incoming new width to call updateWidth()
       */
      self.updateWidthHandler = function (width) {
        self.updateWidth(width)
      }

      /**
       * Exposing method to shrink/redraw grid via compact() function
       * @param {Object} layout
       */
      self.compactHandler = function (layout) {
        self.compact(layout)
      }

      self.setDraggableHandler = function (isDraggable) {
        if (self.isDraggable === null) {
          self.draggable = isDraggable
        }
      }

      self.setResizableHandler = function (isResizable) {
        if (self.isResizable === null) {
          self.resizable = isResizable
        }
      }

      self.setRowHeightHandler = function (rowHeight) {
        self.rowHeight = rowHeight;
      }

      self.setColNum = colNum => {
        self.cols = parseInt(colNum);
      }

      this.eventBus.$on('updateWidth', self.updateWidthHandler);
      this.eventBus.$on('compact', self.compactHandler);
      this.eventBus.$on('setDraggable', self.setDraggableHandler);
      this.eventBus.$on('setResizable', self.setResizableHandler);
      this.eventBus.$on('setRowHeight', self.setRowHeightHandler);
      this.eventBus.$on('setColNum', self.setColNum);
    },
    beforeDestroy: function () {
      var self = this;
      // Remove listeners
      this.eventBus.$off('updateWidth', self.updateWidthHandler);
      this.eventBus.$off('compact', self.compactHandler);
      this.eventBus.$off('setDraggable', self.setDraggableHandler);
      this.eventBus.$off('setResizable', self.setResizableHandler);
      this.eventBus.$off('setRowHeight', self.setRowHeightHandler);
      this.eventBus.$off('setColNum', self.setColNum);
    },
    mounted: function () {
      // @todo - this looks at this.$parent, should pass in?
      this.cols = this.$parent.colNum
      this.rowHeight = this.$parent.rowHeight
      this.containerWidth = this.$parent.width !== null ? this.$parent.width : 100
      this.margin = this.$parent.margin !== undefined ? this.$parent.margin : [10, 10]
      this.maxRows = this.$parent.maxRows

      if (this.isDraggable === null) {
        this.draggable = this.$parent.isDraggable
      } else {
        this.draggable = this.isDraggable
      }

      if (this.isResizable === null) {
        this.resizable = this.$parent.isResizable
      } else {
        this.resizable = this.isResizable
      }

      this.useCssTransforms = this.$parent.useCssTransforms
      this.createStyle()
    },
    watch: {
      isDraggable: function () {
        this.draggable = this.isDraggable
      },
      draggable: function () {
        var self = this
        if (this.interactObj === null || this.interactObj === undefined) {
          this.interactObj = interact(this.$refs.item)
        }
        if (this.draggable) {
          var opts = {
            ignoreFrom: this.dragIgnoreFrom,
            allowFrom: this.dragAllowFrom
          }
          this.interactObj.draggable(opts)
          if (!this.dragEventSet) {
            this.dragEventSet = true
            this.interactObj.on('dragstart dragmove dragend', function (event) {
              self.handleDrag(event)
            })
          }
        } else {
          this.interactObj.draggable({
            enabled: false
          })
        }
      },
      isResizable: function () {
        this.resizable = this.isResizable
      },
      resizable: function () {
        var self = this
        if (this.interactObj === null || this.interactObj === undefined) {
          this.interactObj = interact(this.$refs.item)
        }
        if (this.resizable) {
          var opts = {
            preserveAspectRatio: false,
            edges: {
              left: false,
              right: '.' + this.resizableHandleClass,
              bottom: '.' + this.resizableHandleClass,
              top: false
            },
            ignoreFrom: this.resizeIgnoreFrom
          }

          this.interactObj.resizable(opts);

          if (!this.resizeEventSet) {
            this.resizeEventSet = true
            this.interactObj.on('resizestart resizemove resizeend', function (
              event
            ) {
              self.handleResize(event);
            })
          }
        } else {
          this.interactObj.resizable({
            enabled: false
          })
        }
      },
      rowHeight: function () {
        this.createStyle();
      },
      cols: function () {
        this.createStyle();
      },
      containerWidth: function () {
        this.createStyle();
      },
      x: function (newVal) {
        this.innerX = newVal;
        this.createStyle();
      },
      y: function (newVal) {
        this.innerY = newVal;
        this.createStyle();
      },
      h: function (newVal) {
        this.innerH = newVal;
        this.createStyle();
      },
      w: function (newVal) {
        this.innerW = newVal;
        this.createStyle();
      }
    },
    computed: {
      resizableHandleClass () {
        return 'vue-resizable-handle'
      }
    },
    methods: {
      createStyle: function () {
        if (this.x + this.w > this.cols) {
          this.innerX = 0
          this.innerW = this.w > this.cols ? this.cols : this.w
        } else {
          this.innerX = this.x
          this.innerW = this.w
        }
        var pos = this.calcPosition(
          this.innerX,
          this.innerY,
          this.innerW,
          this.innerH
        )

        if (this.isDragging) {
          pos.top = this.dragging.top
          pos.left = this.dragging.left
        }
        if (this.isResizing) {
          pos.width = this.resizing.width
          pos.height = this.resizing.height
        }

        let style
        // CSS Transforms support (default)
        if (this.useCssTransforms) {
          style = setTransform(pos.top, pos.left, pos.width, pos.height)
        } else {
          // top,left (slow)
          style = setTopLeft(pos.top, pos.left, pos.width, pos.height)
        }
        this.style = style
      },
      handleResize: function (event) {
        const position = getControlPosition(event)
        // Get the current drag point from the event. This is used as the offset.
        if (position == null) return // not possible but satisfies flow
        const {x, y} = position

        const newSize = {width: 0, height: 0}
        switch (event.type) {
          case 'resizestart':
            this.previousW = this.innerW
            this.previousH = this.innerH
            var pos = this.calcPosition(
              this.innerX,
              this.innerY,
              this.innerW,
              this.innerH
            )
            newSize.width = pos.width
            newSize.height = pos.height
            this.resizing = newSize
            this.isResizing = true
            break
          case 'resizemove':
            const coreEvent = createCoreData(this.lastW, this.lastH, x, y)
            newSize.width = this.resizing.width + coreEvent.deltaX
            newSize.height = this.resizing.height + coreEvent.deltaY

            this.resizing = newSize
            break
          case 'resizeend':
            var pos = this.calcPosition(
              this.innerX,
              this.innerY,
              this.innerW,
              this.innerH
            )
            newSize.width = pos.width
            newSize.height = pos.height

            this.resizing = null
            this.isResizing = false
            break
        }

        // Get new WH
        var pos = this.calcWH(newSize.height, newSize.width)
        if (pos.w < this.minW) {
          pos.w = this.minW
        }
        if (pos.w > this.maxW) {
          pos.w = this.maxW
        }
        if (pos.h < this.minH) {
          pos.h = this.minH
        }
        if (pos.h > this.maxH) {
          pos.h = this.maxH
        }

        if (pos.h < 1) {
          pos.h = 1
        }
        if (pos.w < 1) {
          pos.w = 1
        }

        this.lastW = x
        this.lastH = y

        if (this.innerW !== pos.w || this.innerH !== pos.h) {
          this.$emit(
            'resize',
            this.i,
            pos.h,
            pos.w,
            newSize.height,
            newSize.width
          )
        }
        if (
          event.type === 'resizeend' &&
          (this.previousW !== this.innerW || this.previousH !== this.innerH)
        ) {
          this.$emit(
            'resized',
            this.i,
            pos.h,
            pos.w,
            newSize.height,
            newSize.width
          )
        }
        this.eventBus.$emit(
          'resizeEvent',
          event.type,
          this.i,
          this.innerX,
          this.innerY,
          pos.h,
          pos.w
        )
      },
      handleDrag (event) {
        let shouldUpdate;
        let parentRect; // parent bounding box
        let clientRect; // this bounding box
        let pos;

        const position = getControlPosition(event);
        const newPosition = {
          top: 0,
          left: 0
        }
        const {x, y} = position;

        // Quick check to see if it's in middle of resizing
        if (this.isResizing) {
          return;
        }

        shouldUpdate = false
        switch (event.type) {
          case 'dragstart':
            this.previousX = this.innerX
            this.previousY = this.innerY

            parentRect = event.target.offsetParent.getBoundingClientRect()
            clientRect = event.target.getBoundingClientRect()
            newPosition.left = clientRect.left - parentRect.left
            newPosition.top = clientRect.top - parentRect.top
            this.dragging = newPosition
            this.isDragging = true
            break
          case 'dragend':
            if (!this.isDragging) return
            parentRect = event.target.offsetParent.getBoundingClientRect()
            clientRect = event.target.getBoundingClientRect()
            newPosition.left = clientRect.left - parentRect.left
            newPosition.top = clientRect.top - parentRect.top
            this.dragging = null
            this.isDragging = false
            shouldUpdate = true
            break
          case 'dragmove':
            const coreEvent = createCoreData(this.lastX, this.lastY, x, y)
            newPosition.left = this.dragging.left + coreEvent.deltaX
            newPosition.top = this.dragging.top + coreEvent.deltaY
            this.dragging = newPosition
            break
        }

        pos = this.calcXY(newPosition.top, newPosition.left);

        this.lastX = x;
        this.lastY = y;

        if (this.innerX !== pos.x || this.innerY !== pos.y) {
          this.$emit('move', this.i, pos.x, pos.y);
        }
        if (event.type === 'dragend' && (this.previousX !== this.innerX || this.previousY !== this.innerY)) {
          this.$emit('moved', this.i, pos.x, pos.y);
        }
        this.eventBus.$emit(
          'dragEvent',
          event.type,
          this.i,
          pos.x,
          pos.y,
          this.innerH,
          this.innerW
        )
      },
      calcPosition: function (x, y, w, h) {
        const colWidth = this.calcColWidth()

        var out = {
          left: Math.round(colWidth * x + (x + 1) * this.margin[0]),
          top: Math.round(this.rowHeight * y + (y + 1) * this.margin[1]),
          // 0 * Infinity === NaN, which causes problems with resize constriants;
          // Fix this if it occurs.
          // Note we do it here rather than later because Math.round(Infinity) causes deopt
          width: w === Infinity ? w : Math.round(colWidth * w + Math.max(0, w - 1) * this.margin[0]),
          height: h === Infinity ? h : Math.round(this.rowHeight * h + Math.max(0, h - 1) * this.margin[1])
        }

        return out
      },
      /**
       * Translate x and y coordinates from pixels to grid units.
       * @param  {Number} top  Top position (relative to parent) in pixels.
       * @param  {Number} left Left position (relative to parent) in pixels.
       * @return {Object} x and y in grid units.
       */
      calcXY (top, left) {
        const colWidth = this.calcColWidth();

        let x = Math.round((left - this.margin[0]) / (colWidth + this.margin[0]));
        let y = Math.round((top - this.margin[1]) / (this.rowHeight + this.margin[1]));

        // Capping
        x = Math.max(Math.min(x, this.cols - this.innerW), 0);
        y = Math.max(Math.min(y, this.maxRows - this.innerH), 0);

        return {x, y}
      },
      // Helper for generating column width
      /**
       * Generates column width.
       * - ContainerWidth - margin, * cols+1 / cols
       */
      calcColWidth () {
        return (this.containerWidth - this.margin[0] * (this.cols + 1)) / this.cols;
      },

      /**
       * Given a height and width in pixel values, calculate grid units.
       * @param  {Number} height Height in pixels.
       * @param  {Number} width  Width in pixels.
       * @return {Object} w, h as grid units.
       */
      calcWH (height, width) {
        let colWidth = this.calcColWidth();

        let w = Math.round((width + this.margin[0]) / (colWidth + this.margin[0]));
        let h = Math.round((height + this.margin[1]) / (this.rowHeight + this.margin[1]));

        // Capping
        w = Math.max(Math.min(w, this.cols - this.innerX), 0);
        h = Math.max(Math.min(h, this.maxRows - this.innerY), 0);
        return {w, h}
      },
      updateWidth: function (width, colNum) {
        this.containerWidth = width
        if (colNum !== undefined && colNum !== null) {
          this.cols = colNum;
        }
      },
      /**
       * Method to shrink vertical facets; updating the grid.
       */
      compact: function () {
        this.createStyle();
      }
    }
  }
</script>
<style>
    .vue-grid-item:not(.vue-grid-placeholder) {
        background: rgba(252, 225, 214, 0.85);
        border: 1px solid black;
    }

    .vue-grid-item.resizing {
        opacity: 0.9;
    }

    .vue-grid-item.static {
        background: #cce;
    }

    .vue-grid-item .text {
        font-size: 24px;
        text-align: center;
        position: absolute;
        top: 0;
        bottom: 0;
        left: 0;
        right: 0;
        margin: auto;
        height: 100%;
        width: 100%;
    }

    .vue-grid-item .no-drag {
        height: 100%;
        width: 100%;
    }

    .vue-grid-item .minMax {
        font-size: 12px;
    }

    .vue-grid-item .add {
        cursor: pointer;
    }

    .vue-grid-item {
        transition: all 200ms ease;
        transition-property: left, top, right;
    }

    .vue-grid-item.cssTransforms {
        transition-property: transform;
        left: 0;
        right: auto;
    }

    .vue-grid-item.resizing {
        opacity: 0.6;
        z-index: 3;
    }

    .vue-grid-item.vue-draggable-dragging {
        transition: none;
        z-index: 3;
    }

    .vue-grid-item.vue-grid-placeholder {
        background: red;
        opacity: 0.2;
        transition-duration: 100ms;
        z-index: 2;
        -webkit-user-select: none;
        -moz-user-select: none;
        -ms-user-select: none;
        -o-user-select: none;
        user-select: none;
    }

    .vue-grid-item > .vue-resizable-handle {
        position: absolute;
        width: 20px;
        height: 20px;
        bottom: 0;
        right: 0;
        background: url("data:image/svg+xml;base64,PD94bWwgdmVyc2lvbj0iMS4wIiBzdGFuZGFsb25lPSJubyI/Pg08IS0tIEdlbmVyYXRvcjogQWRvYmUgRmlyZXdvcmtzIENTNiwgRXhwb3J0IFNWRyBFeHRlbnNpb24gYnkgQWFyb24gQmVhbGwgKGh0dHA6Ly9maXJld29ya3MuYWJlYWxsLmNvbSkgLiBWZXJzaW9uOiAwLjYuMSAgLS0+DTwhRE9DVFlQRSBzdmcgUFVCTElDICItLy9XM0MvL0RURCBTVkcgMS4xLy9FTiIgImh0dHA6Ly93d3cudzMub3JnL0dyYXBoaWNzL1NWRy8xLjEvRFREL3N2ZzExLmR0ZCI+DTxzdmcgaWQ9IlVudGl0bGVkLVBhZ2UlMjAxIiB2aWV3Qm94PSIwIDAgNiA2IiBzdHlsZT0iYmFja2dyb3VuZC1jb2xvcjojZmZmZmZmMDAiIHZlcnNpb249IjEuMSINCXhtbG5zPSJodHRwOi8vd3d3LnczLm9yZy8yMDAwL3N2ZyIgeG1sbnM6eGxpbms9Imh0dHA6Ly93d3cudzMub3JnLzE5OTkveGxpbmsiIHhtbDpzcGFjZT0icHJlc2VydmUiDQl4PSIwcHgiIHk9IjBweCIgd2lkdGg9IjZweCIgaGVpZ2h0PSI2cHgiDT4NCTxnIG9wYWNpdHk9IjAuMzAyIj4NCQk8cGF0aCBkPSJNIDYgNiBMIDAgNiBMIDAgNC4yIEwgNCA0LjIgTCA0LjIgNC4yIEwgNC4yIDAgTCA2IDAgTCA2IDYgTCA2IDYgWiIgZmlsbD0iIzAwMDAwMCIvPg0JPC9nPg08L3N2Zz4=");
        background-position: bottom right;
        padding: 0 3px 3px 0;
        background-repeat: no-repeat;
        background-origin: content-box;
        box-sizing: border-box;
        cursor: se-resize;
    }

    .vue-grid-item.disable-userselect {
        user-select: none;
    }
</style>
