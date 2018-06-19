<template>
    <div
            ref="item"
            :style="mergedStyle"
            class="vue-grid-layout">
        <!-- slot for any grid header stuff.  @todo better naming -->
        <slot></slot>
        <!-- placeholder facet to show the ghost image when resizing a facet.-->
        <grid-item class="vue-grid-placeholder"
                   v-show="isDragging"
                   :x="placeholder.x"
                   :y="placeholder.y"
                   :w="placeholder.w"
                   :h="placeholder.h"
                   :i="placeholder.i"></grid-item>
    </div>
</template>
<style>
    .columns {
        -moz-columns: 120px;
        -webkit-columns: 120px;
        columns: 120px;
    }
    .vue-grid-layout {
        background: rgba(252, 225, 214, 0.5);
        position: relative;
        transition: height 200ms ease;
    }
</style>
<script>
  import Vue from 'vue'

  // For smoother scrolling, 3rd party lib.
  let elementResizeDetectorMaker = require('element-resize-detector');

  import {
    bottom,
    compact,
    getLayoutItem,
    moveElement,
    validateLayout
  } from '../utils/utils.js'
  import GridItem from './GridItem.vue'

  export default {
    name: 'GridLayout',
    provide () {
      return {
        eventBus: null
      }
    },
    components: {
      GridItem
    },
    props: {
      /**
       * If true, the container height swells and contracts to fit contents
       */
      autoSize: {
        type: Boolean,
        default: true
      },
      /**
       * Arbitrary number that is divided by viewport width, and that's the column.
       */
      colNum: {
        type: Number,
        default: 12
      },
      /**
       * Pixel height of each row.
       */
      rowHeight: {
        type: Number,
        default: 150
      },
      /**
       * Maximum Number of Rows
       * Default to Infinity
       */
      maxRows: {
        type: Number,
        default: Infinity
      },
      margin: {
        type: Array,
        default: function () {
          return [10, 10]
        }
      },
      /**
       * Boolean if all facets are draggable.
       */
      isDraggable: {
        type: Boolean,
        default: true
      },
      /**
       * Boolean if all facets are resizable.
       */
      isResizable: {
        type: Boolean,
        default: true
      },
      /**
       * todo remove this?
       */
      useCssTransforms: {
        type: Boolean,
        default: true
      },
      /**
       * todo remove this?
       */
      verticalCompact: {
        type: Boolean,
        default: true
      },
      /**
       * THe actual content, facets.
       */
      layout: {
        type: Array,
        required: true
      }
    },
    data: function () {
      return {
        width: null,
        mergedStyle: {},
        lastLayoutLength: 0,
        isDragging: false,
        placeholder: {
          x: 0,
          y: 0,
          w: 0,
          h: 0,
          i: -1
        }
      }
    },
    created () {
      let self = this;

      // Accessible refernces of functions for removing in beforeDestroy
      self.resizeEventHandler = function (eventType, i, x, y, h, w) {
        self.resizeEvent(eventType, i, x, y, h, w)
      }

      self.dragEventHandler = function (eventType, i, x, y, h, w) {
        self.dragEvent(eventType, i, x, y, h, w)
      }

      /**
       * Creating separate Vue for EventBus holding.
       * Attaches to root, used as bypass for all internal events.
       * @event resizeEvent
       * @event dragEvent
       */
      self._provided.eventBus = new Vue();
      self.eventBus = self._provided.eventBus;
      self.eventBus.$on('resizeEvent', self.resizeEventHandler);
      self.eventBus.$on('dragEvent', self.dragEventHandler);
    },
    beforeDestroy: function () {
      // Remove listeners
      this.eventBus.$off('resizeEvent', self.resizeEventHandler)
      this.eventBus.$off('dragEvent', self.dragEventHandler)
      window.removeEventListener('resize', self.onWindowResize)
    },
    mounted: function () {
      this.$nextTick(function () {
        validateLayout(this.layout);
        let self = this;
        this.$nextTick(function () {
          if (self.width === null) {
            self.onWindowResize()
            window.addEventListener('resize', self.onWindowResize)
          }
          compact(self.layout, self.verticalCompact)

          self.updateHeight()
          self.$nextTick(function () {
            var erd = elementResizeDetectorMaker({
              strategy: 'scroll' //<- For ultra performance.
            })
            erd.listenTo(self.$refs.item, function (element) {
              self.onWindowResize()
            })
          })
        })
        window.onload = function () {
          if (self.width === null) {
            self.onWindowResize()
            window.addEventListener('resize', self.onWindowResize)
          }
          compact(self.layout, self.verticalCompact)

          self.updateHeight()
          self.$nextTick(function () {
            var erd = elementResizeDetectorMaker({
              strategy: 'scroll' //<- For ultra performance.
            })
            erd.listenTo(self.$refs.item, function (element) {
              self.onWindowResize()
            })
          })
        }
      })
    },
    watch: {
      /**
       * When width changes the eventBus $emits updateWidth with new Width, then updates Height.
       * @event updateWidth - Event with new Width
       */
      width: function () {
        this.$nextTick(function () {
          this.eventBus.$emit('updateWidth', this.width)
          this.updateHeight()
        })
      },
      layout: function () {
        this.layoutUpdate()
      },
      /**
       * When Column Numbers Change it emits the new Column Number
       * @event setColNum
       */
      colNum: function (val) {
        this.eventBus.$emit('setColNum', val)
      },
      /**
       * When Row Height changes it emits the new Row Height Number
       * @event setRowHeight
       */
      rowHeight: function () {
        this.eventBus.$emit('setRowHeight', this.rowHeight)
      },
      /**
       * If you change the draggable aspect this will emit the new boolean
       * @event setDraggable
       */
      isDraggable: function () {
        this.eventBus.$emit('setDraggable', this.isDraggable)
      },
      /**
       * If you change the resizable aspect this will emit the new boolean
       * @event setResizable
       */
      isResizable: function () {
        this.eventBus.$emit('setResizable', this.isResizable)
      }
    },
    methods: {
      layoutUpdate () {
        if (this.layout !== undefined) {
          if (this.layout.length !== this.lastLayoutLength) {
            this.lastLayoutLength = this.layout.length
          }
          /**
           * @todo check on this; this vertically compacts the layout, then re-emits the width; and updates the height.
           */
          compact(this.layout, this.verticalCompact)
          this.eventBus.$emit('updateWidth', this.width)
          this.updateHeight()
        }
      },
      updateHeight: function () {
        this.mergedStyle = {
          height: this.containerHeight()
        }
      },
      /**
       * Checks this.$refs; which is the HTML element in the DOM.
       *
       * If it's rendered it gives the offsetWidth.
       */
      onWindowResize: function () {
        if (
          this.$refs !== null &&
          this.$refs.item !== null &&
          this.$refs.item !== undefined
        ) {
          this.width = this.$refs.item.offsetWidth
        }
      },
      containerHeight: function () {
        if (!this.autoSize) {
          return
        } else {
          return (bottom(this.layout) * (this.rowHeight + this.margin[1]) + this.margin[1] + 'px')
        }
      },
      dragEvent: function (eventName, id, x, y, h, w) {
        if (eventName === 'dragmove' || eventName === 'dragstart') {
          this.placeholder.i = id
          this.placeholder.x = x
          this.placeholder.y = y
          this.placeholder.w = w
          this.placeholder.h = h
          this.$nextTick(function () {
            this.isDragging = true
          })
          this.eventBus.$emit('updateWidth', this.width)
        } else {
          this.$nextTick(function () {
            this.isDragging = false
          })
        }
        var l = getLayoutItem(this.layout, id)
        // GetLayoutItem sometimes returns null object
        if (l === undefined || l === null) {
          l = {x: 0, y: 0}
        }
        l.x = x
        l.y = y
        // Move the element to the dragged location.
        this.layout = moveElement(this.layout, l, x, y, true)
        compact(this.layout, this.verticalCompact)
        // needed because vue can't detect changes on array element properties
        this.eventBus.$emit('compact')
        this.updateHeight()
        if (eventName === 'dragend') this.$emit('layout-updated', this.layout)
      },
      resizeEvent: function (eventName, id, x, y, h, w) {
        if (eventName === 'resizestart' || eventName === 'resizemove') {
          this.placeholder.i = id
          this.placeholder.x = x
          this.placeholder.y = y
          this.placeholder.w = w
          this.placeholder.h = h
          this.$nextTick(function () {
            this.isDragging = true
          })
          this.eventBus.$emit('updateWidth', this.width)
        } else {
          this.$nextTick(function () {
            this.isDragging = false
          })
        }
        var l = getLayoutItem(this.layout, id)
        // GetLayoutItem sometimes return null object
        if (l === undefined || l === null) {
          l = {h: 0, w: 0}
        }
        l.h = h
        l.w = w
        compact(this.layout, this.verticalCompact)
        this.eventBus.$emit('compact')
        this.updateHeight()
        if (eventName === 'resizeend') this.$emit('layout-updated', this.layout)
      }
    }
  }
</script>
