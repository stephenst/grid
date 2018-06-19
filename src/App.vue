<template>
    <div id="app">
        <grid-layout
            :layout="layout"
            :col-num="parseInt(colNum)"
            :row-height="rowHeight"
            :is-draggable="draggable"
            :is-resizable="resizable"
            :vertical-compact="true"
            :use-css-transforms="true">
            <grid-item v-for="item in layout" :key="item.i"
                       :x="item.x"
                       :y="item.y"
                       :w="item.w"
                       :h="item.h"
                       :min-w="2"
                       :min-h="2"
                       :i="item.i"
                       @resize="resize"
                       @move="move"
                       @resized="resized"
                       @moved="moved">
                <test-element :text="item.i"></test-element>
            </grid-item>
        </grid-layout>
    </div>
</template>

<script>
    import GridItem from './components/GridItem.vue'
    import GridLayout from './components/GridLayout.vue'
    import TestElement from './components/TestElement.vue'

    let testLayout = [
        {'x': 0, 'y': 0, 'w': 2, 'h': 2, 'i': '0', resizable: true, draggable: true},
        {'x': 2, 'y': 0, 'w': 2, 'h': 4, 'i': '1', resizable: null, draggable: null},
        {'x': 4, 'y': 0, 'w': 2, 'h': 5, 'i': '2', resizable: false, draggable: false},
        {'x': 6, 'y': 0, 'w': 2, 'h': 3, 'i': '3', resizable: false, draggable: false},
        {'x': 8, 'y': 0, 'w': 2, 'h': 3, 'i': '4', resizable: false, draggable: false},
        {'x': 10, 'y': 0, 'w': 2, 'h': 3, 'i': '5', resizable: false, draggable: false},
        {'x': 0, 'y': 5, 'w': 2, 'h': 5, 'i': '6', resizable: false, draggable: false},
        {'x': 2, 'y': 5, 'w': 2, 'h': 5, 'i': '7', resizable: false, draggable: false},
        {'x': 4, 'y': 5, 'w': 2, 'h': 5, 'i': '8', resizable: false, draggable: false},
        {'x': 6, 'y': 4, 'w': 2, 'h': 4, 'i': '9', resizable: false, draggable: false},
        {'x': 8, 'y': 4, 'w': 2, 'h': 4, 'i': '10', resizable: false, draggable: false},
        {'x': 10, 'y': 4, 'w': 2, 'h': 4, 'i': '11', resizable: false, draggable: false},
        {'x': 0, 'y': 10, 'w': 2, 'h': 5, 'i': '12', resizable: false, draggable: false},
        {'x': 2, 'y': 10, 'w': 2, 'h': 5, 'i': '13', resizable: false, draggable: false},
        {'x': 4, 'y': 8, 'w': 2, 'h': 4, 'i': '14', resizable: false, draggable: false},
        {'x': 6, 'y': 8, 'w': 2, 'h': 4, 'i': '15', resizable: false, draggable: false},
        {'x': 8, 'y': 10, 'w': 2, 'h': 5, 'i': '16', resizable: false, draggable: false},
        {'x': 10, 'y': 4, 'w': 2, 'h': 2, 'i': '17', resizable: false, draggable: false},
        {'x': 0, 'y': 9, 'w': 2, 'h': 3, 'i': '18', resizable: false, draggable: false},
        {'x': 2, 'y': 6, 'w': 2, 'h': 2, 'i': '19', resizable: false, draggable: false}
    ]

    export default {
        name: 'app',
        components: {
            GridItem,
            GridLayout,
            TestElement
        },
        data () {
            return {
                layout: JSON.parse(JSON.stringify(testLayout)),
                layout2: JSON.parse(JSON.stringify(testLayout)),
                draggable: true,
                resizable: true,
                rowHeight: 30,
                colNum: 12,
                index: 0
            }
        },
        mounted: function () {
            this.index = this.layout.length
        },
        methods: {
            move: function (i, newX, newY) {
                console.log('MOVE i=' + i + ', X=' + newX + ', Y=' + newY);
            },
            resize: function (i, newH, newW) {
                console.log('RESIZE i=' + i + ', H=' + newH + ', W=' + newW);
            },
            moved: function (i, newX, newY) {
                console.log('### MOVED i=' + i + ', X=' + newX + ', Y=' + newY);
            },
            resized: function (i, newH, newW, newHPx, newWPx) {
                console.log('### RESIZED i=' + i + ', H=' + newH + ', W=' + newW + ', H(px)=' + newHPx + ', W(px)=' + newWPx);
            }
        }
    }
</script>

<style>
    #app {
        font-family: 'Avenir', Helvetica, Arial, sans-serif;
        -webkit-font-smoothing: antialiased;
        -moz-osx-font-smoothing: grayscale;
        text-align: center;
        color: #2c3e50;
        margin-top: 60px;
    }
</style>
