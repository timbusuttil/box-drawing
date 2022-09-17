<template>
  <div class="box-renderer" @mouseleave="hoverCell(-1)">
    <pre
      v-for="(cell, i) in currentData" :key="i"
      :class="cellClasses(i)"
      @pointerdown="selectCell(i)"
      @mouseover="hoverCell(i)"
    >{{ cell }}{{ needsNewline(i) ? '\n' : null }}</pre>
  </div>
</template>

<script>
export default {
  name: 'BoxCanvas',
  props: {
    width: Number,
    height: Number,
    startData: Array,
  },
  data() {
    return {
      mode: 'text',
      currentData: [],
      hoveredCell: -1,
      selectedCell: -1,
    }
  },
  methods: {
    needsNewline(i) {
      return i % this.width === this.width - 1;
    },
    cellClasses(i) {
      let classes = [];
      if (this.hoveredCell === i) classes.push('hovered');
      if (this.selectedCell === i) classes.push('selected');
      return classes;
    },
    selectCell(i) {
      this.selectedCell = i;
    },
    hoverCell(i) {
      this.hoveredCell = i;
    },
    handleKey(e) {
      console.log(e.key);
      // toggle modes
      if (e.key === 'Tab') {
        if (this.mode === 'borders') {
          this.mode = 'text';
        } else if (this.mode === 'text') {
          this.mode = 'borders';
        }
        e.preventDefault();
      } else

      // nav
      if (e.key === 'ArrowUp') { this.arrowNav('n') } else
      if (e.key === 'ArrowDown') { this.arrowNav('s') } else
      if (e.key === 'ArrowLeft') { this.arrowNav('w') } else
      if (e.key === 'ArrowRight') { this.arrowNav('e') } else
      if (e.key === 'Backspace') {
        e.preventDefault();
        this.applyInput('DEL');
        this.arrowNav('w')
      } else

      // characters
      if (this.mode === 'borders') {
        if (e.key === 'q') { this.applyInput('╭') } else
        if (e.key === 'w') { this.applyInput('─') } else
        if (e.key === 'e') { this.applyInput('╮') } else
        if (e.key === 'a') { this.applyInput('│') } else
        if (e.key === 's') { this.applyInput('·') } else
        if (e.key === 'd') { this.applyInput('│') } else
        if (e.key === 'z') { this.applyInput('╰') } else
        if (e.key === 'x') { this.applyInput('─') } else
        if (e.key === 'c') { this.applyInput('╯') }
      } else if (this.mode ===  'text') {
        if (e.key.length === 1) this.applyInput(e.key);
      }
    },
    applyInput(input) {
      let parsedInput;
      if (input === 'DEL') {
        console.log('hello');
        parsedInput = ' ';
      } else {
        parsedInput = input;
      }

      if (this.selectedCell !== -1) {
        console.log(parsedInput);
        this.currentData[this.selectedCell] = parsedInput;
        if (this.mode === 'text' && input !== 'DEL') this.arrowNav('e');
      } else if (this.hoveredCell !== -1) {
        this.currentData[this.hoveredCell] = parsedInput;
      }
    },
    arrowNav(direction) {
      let dir = direction[0];
      let dist = 1;
      if (direction.length > 1) dist = direction[1];
      if (dir === 'n') this.selectedCell -= this.width * dist;
      if (dir === 's') this.selectedCell += this.width * dist;
      if (dir === 'w') this.selectedCell -= dist;
      if (dir === 'e') this.selectedCell += dist;
      // clamp
      this.selectedCell = Math.min(Math.max(this.selectedCell, 0), this.currentData.length - 1);
    }
  },
  mounted() {
    // init data
    if (this.startData) {
      this.currentData = this.startData;
    } else {
      let vals = [];
      for (var i = 0; i < this.width * this.height; i++) {
        vals.push(' ')
      }
      this.currentData = [...vals];
    }

    // key event listener
    document.addEventListener('keydown', this.handleKey)
  }
}
</script>

<style scoped>
.box-renderer {
  border: 1px solid red;
  width: fit-content;
}

pre {
  font-family: 'Menlo';
  font-size: 50px;
  width: fit-content;
  margin: 0;
  display: inline;
  user-select: none;
  border: 0.1px solid red;
}

pre.hovered {
  background: rgba(0, 0, 255, 0.1);
}

pre.selected {
  background: rgba(0, 0, 255, 0.25) !important;
}
</style>
