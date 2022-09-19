<template>
  <!-- canvas -->
  <div class="box-renderer" @mouseleave="hoverCell(-1)">
    <pre
      v-for="(cell, i) in currentData" :key="i"
      :class="cellClasses(i)"
      :style="cellStyle"
      @pointerdown="selectCell(i, $event)"
      @mouseover="hoverCell(i, $event)"
    >{{ cell }}{{ needsNewline(i) ? '\n' : null }}</pre>
  </div>
  <!-- controls -->
  <div class="controls">
    <div class="control-group">
      <label><input type="color" v-model="colours.bg">bg col</label>
      <label><input type="color" v-model="colours.fg">fg col</label>
      <label><input type="checkbox" v-model="colours.flipped">flip fg/bg</label>
      <button @click="setColorsFromPresets">random preset</button>
    </div>
    <div class="control-group">
      <label><input type="checkbox" v-model="showBorders">show cell borders</label>
    </div>
  </div>
</template>

<script>
import { presets } from '@/assets/colours.js';

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
      selectedCells: [],
      lastSelected: -1,
      showBorders: false,
      colours: {
        bg: '#000000',
        fg: '#0000ff',
        flipped: false,
      }
    }
  },
  computed: {
    cellStyle() {
      let cols = [this.colours.bg, this.colours.fg];
      if (this.colours.flipped) cols = cols.reverse();
      return {
        background: cols[0],
        color: cols[1]
      }
    }
  },
  methods: {
    needsNewline(i) {
      return i % this.width === this.width - 1;
    },
    cellClasses(i) {
      let classes = [];
      if (this.hoveredCell === i) classes.push('hovered');
      if (this.selectedCells.includes(i)) classes.push('selected');
      if (this.showBorders) classes.push('borders')
      return classes;
    },
    selectCell(i, e) {
      this.lastSelected = i;
      if (e.shiftKey) {
        this.selectedCells.push(i);
      } else {
        this.selectedCells = [i];
      }
    },
    selectArea(cell1, cell2) {
      this.selectedCells = [];
      const co1 = this.coordsByCell(cell1);
      const co2 = this.coordsByCell(cell2);
      const sX = Math.min(co1[0], co2[0]);
      const sY = Math.min(co1[1], co2[1]);
      const eX = Math.max(co1[0], co2[0]);
      const eY = Math.max(co1[1], co2[1]);
      const diffX = eX - sX;
      const diffY = eY - sY;
      for (let x = 0; x <= diffX; x++) {
        for (let y = 0; y <= diffY; y++) {
          this.selectedCells.push(this.cellByCoords(sX + x, sY + y))
        }
      }
    },
    hoverCell(i, e) {
      if (e && e.buttons > 0) {
        this.selectArea(this.lastSelected, i);
      } else {
        this.hoveredCell = i;
      }
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

      // nav/controls
      if (e.key === 'ArrowUp') { this.arrowNav('n') } else
      if (e.key === 'ArrowDown') { this.arrowNav('s') } else
      if (e.key === 'ArrowLeft') { this.arrowNav('w') } else
      if (e.key === 'ArrowRight') { this.arrowNav('e') } else
      if (e.key === 'Escape') { this.selectedCells = [] } else
      if (e.key === 'b' && e.metaKey) { this.boxShortcut() } else
      if (e.key === 'x' && e.metaKey) { this.cut() } else
      if (e.key === 'c' && e.metaKey) { this.copy() } else
      if (e.key === 'v' && e.metaKey) { this.paste() } else
      if (e.key === 'Enter') { this.newline() } else
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
    applyInput(input, targetCells = []) {
      let parsedInput;
      if (input === 'DEL') {
        parsedInput = ' ';
      } else {
        parsedInput = input;
      }

      if (targetCells.length > 0) {
        targetCells.forEach((cell) => {
          this.currentData[cell] = parsedInput;
        });
      } else {
        if (this.selectedCells.length > 0) {
          this.selectedCells.forEach((cell) => {
            this.currentData[cell] = parsedInput;
          });
          if (this.mode === 'text' && input !== 'DEL') this.arrowNav('e');
        } else if (this.hoveredCell !== -1) {
          this.currentData[this.hoveredCell] = parsedInput;
        }
      }
    },
    arrowNav(direction) {
      let dir = direction[0];
      let dist = 1;
      if (direction.length > 1) dist = direction[1];
      if (this.selectedCells.length === 1) {
        if (dir === 'n') this.selectedCells[0] -= this.width * dist;
        if (dir === 's') this.selectedCells[0] += this.width * dist;
        if (dir === 'w') this.selectedCells[0] -= dist;
        if (dir === 'e') this.selectedCells[0] += dist;
        // clamp
        this.selectedCells[0] = Math.min(Math.max(this.selectedCells[0], 0), this.currentData.length - 1);
      }
    },
    coordsByCell(i) {
      return [
        i % this.width,
        Math.floor(i / this.width)
      ];
    },
    cellByCoords(x, y) {
      return x + y * this.width;
    },
    setColorsFromPresets() {
      let i = Math.floor(Math.random() * presets.length);
      console.log(i);
      console.log(presets[i]);
      console.log(this.colours.fg);
      this.colours.fg = presets[i].fg;
      this.colours.bg = presets[i].bg;
    },
    boxShortcut() {
      console.log('todo: box!');
      if (this.selectedCells.length > 1) {
        // get min & max width & height
        let minX = 999;
        let minY = 999;
        let maxX = -1;
        let maxY = -1;
        this.selectedCells.forEach((cell) => {
          const coords = this.coordsByCell(cell);
          if (coords[0] < minX) minX = coords[0];
          if (coords[0] < minY) minY = coords[1];
          if (coords[0] > maxX) maxX = coords[0];
          if (coords[0] > maxY) maxY = coords[1];
        });
        const boxWidth = maxX - minX;
        const boxHeight = maxY - minY;

        // if 1 cell vertical but >1 horizontal, draw row?
        // if 1 cell horizontal but >1 vertical, draw column?
        // if >1 both, draw box
        if (boxWidth > 1 &&  boxHeight > 1) {
          // corners
          this.applyInput('╭', [this.cellByCoords(minX, minY)]);
          this.applyInput('╮', [this.cellByCoords(maxX, minY)]);
          this.applyInput('╰', [this.cellByCoords(minX, maxY)]);
          this.applyInput('╯', [this.cellByCoords(maxX, maxY)]);
          // top/bottom
          let hCells = [];
          for (let x = 1; x < boxWidth; x++) {
            hCells.push(this.cellByCoords(minX + x, minY));
            hCells.push(this.cellByCoords(minX + x, maxY));
          }
          this.applyInput('─', hCells);
          // left/right
          let vCells = [];
          for (let y = 1; y < boxHeight; y++) {
            vCells.push(this.cellByCoords(minX, minY + y));
            vCells.push(this.cellByCoords(maxX, minY + y));
          }
          this.applyInput('│', vCells);
        }
      }
    },
    cut() {
      console.log('todo: cut!');
    },
    copy() {
      console.log('todo: copy!');
    },
    paste() {
      console.log('todo: paste!');
    },
    newline() {
      this.selectedCells = [this.lastSelected + this.width];
      this.lastSelected = this.selectedCells[0];
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

    // random cols
    this.setColorsFromPresets();

    // key event listener
    document.addEventListener('keydown', this.handleKey)
  },
  unmounted() {
    document.removeEventListener('keydown', this.handleKey)
  }
}
</script>

<style scoped>
/* ----------------------------------------------------------- CANVAS */
.box-renderer {
  border: 1px solid rgba(0, 0, 0, 0.25);
  width: fit-content;
  border-radius: 8px;
  overflow: hidden;
}

/* ----------------------------------------------------------- CELLS */
pre {
  font-family: 'Menlo';
  font-size: 50px;
  width: fit-content;
  margin: 0;
  display: inline;
  user-select: none;
  box-sizing: border-box;
}

pre.borders {
  box-shadow: inset 0 0 0 1px rgba(0, 0, 0, 0.25);
}

pre.hovered {
  box-shadow: inset 0 0 0 50px rgba(0, 0, 0, 0.1) !important;
}

pre.selected {
  box-shadow: inset 0 0 0 50px rgba(0, 0, 0, 0.25) !important;
}

/* ----------------------------------------------------------- CONTROLS */
.controls {
  display: flex;
}

.control-group {
  padding: 10px 0;
  display: flex;
  flex-direction: column;
  width: 140px;
}

.control-group > * {
  margin: 3px 0;
  padding: 2px 0;
}

label {
  user-select: none;
}

input {
  margin-right: 5px;
}
</style>
