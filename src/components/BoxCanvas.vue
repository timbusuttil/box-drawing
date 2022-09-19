<template>
  <!-- canvas -->
  <div id="box-renderer" :style="rendererStyle" @mouseleave="hoverCell(-1)">
    <pre
      v-for="(cell, i) in currentData" :key="i"
      :class="cellClasses(i)"
      :style="cellStyle"
      @pointerdown="selectCell(i, $event)"
      @mouseover="hoverCell(i, $event)"
    >{{ cell }}{{ needsNewline(i) ? '\n' : null }}</pre>
  </div>
  <!-- controls -->
  <div class="controls" @click="selectedCells = []">
    <div class="control-group">
      <p>colours</p>
      <label><input type="color" v-model="colours.bg">bg col</label>
      <label><input type="color" v-model="colours.fg">fg col</label>
      <label><input type="checkbox" v-model="colours.flipped">flip fg/bg</label>
      <button @click="setColorsFromPresets">random preset</button>
    </div>
    <div class="control-group">
      <p>debug</p>
      <label><input type="checkbox" v-model="showBorders">show cell borders</label>
    </div>
    <div class="control-group">
      <p>saving</p>
      <label><input type="text" v-model="image.name">name</label>
      <label><input type="number" v-model="image.scale">scale</label>
      <button @click="saveImage">save image</button>
    </div>
  </div>
</template>

<script>
import { nextTick } from 'vue';
import { presets } from '@/assets/colours.js';
import html2canvas from 'html2canvas';

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
      clipboard: {
        x: -1,
        y: -1,
        contents: []
      },
      showBorders: false,
      colours: {
        bg: '#000000',
        fg: '#0000ff',
        flipped: false,
        lastPreset: -1,
      },
      image: {
        name: 'untitled',
        scale: 2,
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
    },
    rendererStyle() {
      let cols = [this.colours.bg, this.colours.fg];
      if (this.colours.flipped) cols = cols.reverse();
      return {
        background: cols[0]
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
      // console.log(e.key);
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
        this.applyInput('DEL', [], this.selectedCells.length === 1);
      } else

      // characters
      if (this.mode === 'borders') {
        if (e.key === 'q') { this.applyInput('╭') } else
        if (e.key === 'w') { this.applyInput('╮') } else
        if (e.key === 'a') { this.applyInput('╰') } else
        if (e.key === 's') { this.applyInput('╯') } else
        if (e.key === 'e') { this.applyInput('│') } else
        if (e.key === 'd') { this.applyInput('─') }
      } else if (this.mode ===  'text') {
        if (e.key.length === 1) this.applyInput(e.key, [], this.selectedCells.length === 1);
      }
    },
    applyInput(input, targetCells = [], autoAdvance = false) {
      let parsedInput;
      if (input === 'DEL') {
        parsedInput = ' ';
      } else {
        parsedInput = input;
      }

      if (targetCells.length > 0) {
        targetCells.forEach((cell) => {
          if (cell > 0 && cell < this.currentData.length) this.currentData[cell] = parsedInput;
        });
      } else {
        if (this.selectedCells.length > 0) {
          this.selectedCells.forEach((cell) => {
            if (cell > 0 && cell < this.currentData.length) this.currentData[cell] = parsedInput;
          });
          if (autoAdvance) {
            if (input === 'DEL') {
              this.arrowNav('w');
            } else {
              this.arrowNav('e');
            }
          }
        } else if (this.hoveredCell !== -1) {
          this.currentData[this.hoveredCell] = parsedInput;
        }
      }
    },
    arrowNav(direction) {
      this.hoveredCell = -1;
      let dir = direction[0];
      let dist = 1;
      if (direction.length > 1) dist = direction[1];
      this.selectedCells.forEach((cell, i) => {
        if (dir === 'n') this.selectedCells[i] -= this.width * dist;
        if (dir === 's') this.selectedCells[i] += this.width * dist;
        if (dir === 'w') this.selectedCells[i] -= dist;
        if (dir === 'e') this.selectedCells[i] += dist;
      });
      // clamp
      if (this.selectedCells.length === 1) {
        this.selectedCells[0] = Math.min(Math.max(this.selectedCells[0], 0), this.currentData.length - 1);
      } else {
        // deal with this case
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
      if (i === this.colours.lastPreset) i++;
      this.colours.lastPreset = i;
      this.colours.fg = presets[i].fg;
      this.colours.bg = presets[i].bg;
    },
    boxShortcut() {
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
      this.copy();
      this.applyInput(' ', [...this.selectedCells]);
    },
    copy() {
      if (this.selectedCells.length > 0) {
        // get coordinates of first point
        const refCoords = this.coordsByCell(this.selectedCells[0]);
        this.clipboard.x = refCoords[0];
        this.clipboard.y = refCoords[1];
        // populate contents array
        this.clipboard.contents = this.selectedCells.map(cell => {
          const coords = this.coordsByCell(cell);
          return {
            xoff: coords[0] - refCoords[0],
            yoff: coords[1] - refCoords[1],
            data: this.currentData[cell]
          }
        })
      }
      console.log(this.clipboard);
    },
    paste() {
      if (this.selectedCells.length > 0 && this.clipboard.x !== -1) {
        this.clipboard.contents.forEach((item) => {
          this.applyInput(item.data, [this.lastSelected + item.xoff + (item.yoff * this.width)]);
        });
      }
    },
    newline() {
      this.selectedCells = [this.lastSelected + this.width];
      this.lastSelected = this.selectedCells[0];
    },
    async saveImage() {
      const el = document.getElementById('box-renderer');
      const border = getComputedStyle(el)['border'];
      const radius = getComputedStyle(el)['border-radius'];
      const selection = this.selectedCells;
      el.style.border = 'none';
      el.style.borderRadius = 0;
      this.selectedCells = [];
      await nextTick();

      html2canvas(el, {
        scale: this.image.scale
      }).then((canvas) => {
        el.style.border = border;
        el.style.borderRadius = radius;
        this.selectedCells = selection;

        let anchor = document.createElement('a');
        anchor.href = canvas.toDataURL('image/png');
        anchor.download = `${this.image.name}.png`;
        anchor.click();
      });
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
#box-renderer {
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
  padding: 8px;
  border-radius: 8px;
  margin: 8px 4px;
  display: flex;
  flex-direction: column;
  width: 140px;
  border: 1px solid rgba(0, 0, 0, 0.25);
}

.control-group:first-of-type {
  margin-left: 0;
}

.control-group:last-of-type {
  margin-right: 0;
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
  width: calc(100% - 50px);
}

input[type='checkbox'] {
  width: unset;
}
</style>
