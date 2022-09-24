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
      <p>canvas</p>
      <label><input type="checkbox" v-model="showBorders">show cell borders</label>
      <label><input class="long-label" type="number" v-model="fontSize">cell size</label>
      <button @click="center('horizontal')">center horizontally</button>
      <button @click="center('vertical')">center vertically</button>
      <div class="two-buttons">
        <button @click="changeCanvasSize('remove', 'top')">-</button>
        <label>top</label>
        <button @click="changeCanvasSize('add', 'top')">+</button>
      </div>
      <div class="two-buttons">
        <button @click="changeCanvasSize('remove', 'bottom')">-</button>
        <label>bottom</label>
        <button @click="changeCanvasSize('add', 'bottom')">+</button>
      </div>
      <div class="two-buttons">
        <button @click="changeCanvasSize('remove', 'left')">-</button>
        <label>left</label>
        <button @click="changeCanvasSize('add', 'left')">+</button>
      </div>
      <div class="two-buttons">
        <button @click="changeCanvasSize('remove', 'right')">-</button>
        <label>right</label>
        <button @click="changeCanvasSize('add', 'right')">+</button>
      </div>

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
import { cloneArray } from '@/utils/utils.js';
import html2canvas from 'html2canvas';

export default {
  name: 'BoxCanvas',
  props: {
    initWidth: {
      type: Number,
      required: false,
      default: 11
    },
    initHeight: {
      type: Number,
      required: false,
      default: 5
    },
    initData: {
      type: Array,
      required: false
    }
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
      fontSize: 50,
      colours: {
        bg: '#000000',
        fg: '#0000ff',
        flipped: false,
        lastPreset: -1,
      },
      image: {
        name: 'untitled',
        scale: 2,
      },
      editBuffer: [],
      bufferPosition: -1,
    }
  },
  computed: {
    cellStyle() {
      let cols = [this.colours.bg, this.colours.fg];
      if (this.colours.flipped) cols = cols.reverse();
      return {
        background: cols[0],
        color: cols[1],
        fontSize: `${this.fontSize}px`
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
    selectArea(cell1, cell2, shift) {
      if (!shift) this.selectedCells = [];
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
      this.selectedCells = this.deduplicate(this.selectedCells);
    },
    deduplicate(array) {
      return [...new Set(array)];
    },
    hoverCell(i, e) {
      if (e && e.buttons > 0) {
        this.selectArea(this.lastSelected, i, e.shiftKey);
      }
      this.hoveredCell = i;
    },
    handleClick(e) {
      if (e.target.tagName !== 'PRE') this.selectedCells = [];
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
      if (e.key === 'ArrowUp') { this.arrowNav('n', true, e) } else
      if (e.key === 'ArrowDown') { this.arrowNav('s', true, e) } else
      if (e.key === 'ArrowLeft') { this.arrowNav('w', true, e) } else
      if (e.key === 'ArrowRight') { this.arrowNav('e', true, e) } else
      if (e.key === 'Escape') { this.selectedCells = [] } else
      if (e.key === 'a' && e.metaKey) { e.preventDefault(); this.selectAll() } else
      if (e.key === 'b' && e.metaKey) { this.boxShortcut() } else
      if (e.key === 'x' && e.metaKey) { this.cut() } else
      if (e.key === 'c' && e.metaKey) { this.copy() } else
      if (e.key === 'v' && e.metaKey) { this.paste() } else
      if (e.key === 'z' && e.metaKey && !e.shiftKey) { this.undo() } else
      if (e.key === 'y' && e.metaKey) { e.preventDefault(); this.redo() } else
      if (e.key === 'z' && e.metaKey && e.shiftKey) { this.redo() } else
      if (e.key === 'Enter') { this.newline() } else
      if (e.key === 'Backspace') { e.preventDefault(); this.applyInput('DEL', [], this.selectedCells.length === 1) } else

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
    applyInput(input, targetCells = [], autoAdvance = false, useLastBufferIndex = false) {
      const bufferIndex = useLastBufferIndex ? this.bufferPosition : this.bufferPosition + 1;
      const trimBufferAfter = !useLastBufferIndex;
      let parsedInput;
      if (input === 'DEL') {
        parsedInput = ' ';
      } else {
        parsedInput = input;
      }

      if (targetCells.length > 0) {
        targetCells.forEach((cell) => {
          if (cell > -1 && cell < this.currentData.length) {
            this.pushToEditBuffer(bufferIndex, cell, this.currentData[cell], parsedInput, trimBufferAfter);
            this.currentData[cell] = parsedInput;
          }
        });
      } else {
        if (this.selectedCells.length > 0) {
          this.selectedCells.forEach((cell) => {
            if (cell > -1 && cell < this.currentData.length) {
              this.pushToEditBuffer(bufferIndex, cell, this.currentData[cell], parsedInput, trimBufferAfter);
              this.currentData[cell] = parsedInput;
            }
          });
          if (autoAdvance) {
            if (input === 'DEL') {
              this.arrowNav('w');
            } else {
              this.arrowNav('e');
            }
          }
        } else if (this.hoveredCell !== -1) {
          this.pushToEditBuffer(bufferIndex, this.hoveredCell, this.currentData[this.hoveredCell], parsedInput, trimBufferAfter);
          this.currentData[this.hoveredCell] = parsedInput;
        }
      }
    },
    arrowNav(direction, standardInput = false, event = null) {
      this.hoveredCell = -1;
      let dir = direction[0];
      let dist = 1;
      if (direction.length > 1) dist = direction[1];

      if (event) event.preventDefault();

      // remember to also require standardInput = true to avoid weirdness with capitals
      // todo: support meta
      // todo: support alt

      this.selectedCells.forEach((cell, i) => {
        if (dir === 'n') this.selectedCells[i] -= this.width * dist;
        if (dir === 's') this.selectedCells[i] += this.width * dist;
        if (dir === 'w') this.selectedCells[i] -= dist;
        if (dir === 'e') this.selectedCells[i] += dist;
      });

      // todo: support shift

      // clamp
      if (this.selectedCells.length === 1) {
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
      if (i === this.colours.lastPreset) i++;
      this.colours.lastPreset = i;
      this.colours.fg = presets[i].fg;
      this.colours.bg = presets[i].bg;
    },
    selectAll() {
      this.selectedCells = this.currentData.map((cell, i) => i)
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
          if (coords[1] < minY) minY = coords[1];
          if (coords[0] > maxX) maxX = coords[0];
          if (coords[1] > maxY) maxY = coords[1];
        });
        const boxWidth = maxX - minX;
        const boxHeight = maxY - minY;

        // dummy input for edit buffer
        this.applyInput('!', [this.cellByCoords(minX, minY)]);
        // top/bottom
        if (boxWidth >= 1) {
          let hCells = [];
          for (let x = 0; x <= boxWidth; x++) {
            hCells.push(this.cellByCoords(minX + x, minY));
            hCells.push(this.cellByCoords(minX + x, maxY));
          }
          this.applyInput('─', hCells, false, true);
        }
        // left/right
        if (boxHeight >= 1) {
          let vCells = [];
          for (let y = 0; y <= boxHeight; y++) {
            vCells.push(this.cellByCoords(minX, minY + y));
            vCells.push(this.cellByCoords(maxX, minY + y));
          }
          this.applyInput('│', vCells, false, true);
        }
        // corners
        if (boxWidth >= 1 && boxHeight >= 1) {
          this.applyInput('╭', [this.cellByCoords(minX, minY)], false, true);
          this.applyInput('╮', [this.cellByCoords(maxX, minY)], false, true);
          this.applyInput('╰', [this.cellByCoords(minX, maxY)], false, true);
          this.applyInput('╯', [this.cellByCoords(maxX, maxY)], false, true);
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
    },
    paste() {
      if (this.selectedCells.length > 0 && this.clipboard.x !== -1) {
        const startCell = this.selectedCells.length === 1 ? this.selectedCells[0] : this.lastSelected;
        this.clipboard.contents.forEach((item, i) => {
          this.applyInput(item.data, [startCell + item.xoff + (item.yoff * this.width)], false, i !== 0);
        });
      }
    },
    newline() {
      this.selectedCells = [this.lastSelected + this.width];
      this.lastSelected = this.selectedCells[0];
    },
    undo() {
      this.applyBuffer(this.bufferPosition, 'undo');
    },
    redo() {
      this.applyBuffer(this.bufferPosition, 'redo');
    },
    applyBuffer(position, direction, advance = true) {
      let bufferPosition, advanceAmt, applyKey;
      if (direction === 'undo') {
        bufferPosition = position;
        advanceAmt = -1;
        applyKey = 'before';
      } else if (direction === 'redo') {
        bufferPosition = position + 1;
        advanceAmt = +1;
        applyKey = 'after';
      }
      if (this.editBuffer[bufferPosition] !== undefined) {
        const edits = cloneArray(this.editBuffer[bufferPosition]);
        if (direction === 'undo') edits.reverse();
        edits.forEach((edit) => {
          this.currentData[edit.cell] = edit[applyKey];
        });
        if (advance) {
          this.bufferPosition += advanceAmt;
        }
      }
    },
    pushToEditBuffer(bufferIndex, cell, before, after, trimBufferAfter = false) {
      this.bufferPosition = bufferIndex;
      if (trimBufferAfter && bufferIndex < this.editBuffer.length) this.editBuffer.length = bufferIndex;
      if (this.editBuffer[bufferIndex] === undefined) {
        this.editBuffer[bufferIndex] = [];
      }
      if (before !== after) {
        this.editBuffer[bufferIndex].push({ cell, before, after });
      }
    },
    center(direction) {
      // get bounds of non-blank cells
      // and create temp array
      let minX = 999;
      let minY = 999;
      let maxX = -1;
      let maxY = -1;
      let tempArray = [];
      this.currentData.forEach((cell, i) => {
        tempArray.push(' ');
        if (cell !== ' ') {
          const coords = this.coordsByCell(i);
          if (coords[0] < minX) minX = coords[0];
          if (coords[0] < minY) minY = coords[1];
          if (coords[0] > maxX) maxX = coords[0];
          if (coords[0] > maxY) maxY = coords[1];
        }
      });

      // calc distance to move in both directions
      const boxWidth = 1 + maxX - minX;
      const boxHeight = 1 + maxY - minY;
      const distX = minX - Math.floor((this.width - boxWidth) / 2);
      const distY = minY - Math.floor((this.height - boxHeight) / 2);

      // copy contents across based on direction
      if (direction === 'horizontal' || direction === 'both') {
        this.currentData.forEach((cell, i) => {
          const coords = this.coordsByCell(i);
          const target = [coords[0] + distX, coords[1]];
          let newContents = ' ';
          if (target[0] > -1 && target[0] < this.width) newContents = this.currentData[this.cellByCoords(...target)];
          tempArray[i] = newContents;
        });
      }
      if (direction === 'vertical' || direction === 'both') {
        this.currentData.forEach((cell, i) => {
          const coords = this.coordsByCell(i);
          const target = [coords[0], coords[1] + distY];
          let newContents = ' ';
          if (target[1] > -1 && target[1] < this.height) newContents = this.currentData[this.cellByCoords(...target)];
          tempArray[i] = newContents;
        });
      }

      // copy temp array to data
      const bufferIndex = this.editBuffer.length;
      tempArray.forEach((cell, i) => {
        this.pushToEditBuffer(bufferIndex, i, this.currentData[i], cell);
        this.currentData[i] = cell;
      });
      // this.currentData = tempArray;
    },
    changeCanvasSize(mode, side) {
      if (side === 'top' || side === 'bottom') {
        this.height++;
        const extraCells = Array(this.width).fill(' ');
        if (mode === 'add') {
          if (side === 'top') {
            this.currentData.unshift(...extraCells);
          } else {
            this.currentData.push(...extraCells);
          }
        } else if (mode === 'remove') {
          if (side === 'top') {
            this.currentData.splice(0, this.width);
            this.height--;
          } else {
            this.currentData.splice(this.currentData.length - this.width, this.width);
            this.height--;
          }
        }
      } else if (side === 'left' || side === 'right') {
        let x = side === 'left' ? 0 : this.width;
        for (let y = this.height - 1; y >= 0; y--) {
          const newIndex = this.cellByCoords(x, y);
          if (mode === 'add') {
            this.currentData.splice(newIndex, 0, ' ');
          } else if (mode === 'remove') {
            this.currentData.splice(newIndex-1, 1);
          }
        }
        if (mode === 'add') {
          this.width++;
        } else if (mode === 'remove') {
          this.width--;
        }
      }
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
    this.width = this.initWidth;
    this.height = this.initHeight;
    if (this.initData) {
      this.currentData = this.initData;
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
    document.addEventListener('pointerdown', this.handleClick);
  },
  unmounted() {
    document.removeEventListener('keydown', this.handleKey)
    document.removeEventListener('pointerdown', this.handleClick);
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

pre.selected.hovered {
  box-shadow: inset 0 0 0 50px rgba(0, 0, 0, 0.35) !important;
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

div.two-buttons {
  display: flex;
  justify-content: space-around;
  text-align: center;
}

div.two-buttons button {
  width: 28%;
}

div.two-buttons label {
  flex-grow: 1;
}

input {
  margin-right: 5px;
  width: calc(100% - 50px);
}

input.long-label {
  width: calc(100% - 70px);
}

input[type='checkbox'] {
  width: unset;
}

button.short-button {
  width: 40px;
}
</style>
