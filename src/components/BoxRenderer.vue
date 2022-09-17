<template>
  <div class="box-renderer">
    <pre>{{ parsedData }}</pre>
  </div>
</template>

<script>
export default {
  name: 'BoxRenderer',
  props: {
    width: Number,
    startData: Array
  },
  data() {
    return {
      currentData: [],
    }
  },
  computed: {
    parsedData() {
      let result = '';
      this.currentData.forEach((item, i) => {
        if (i !== 0 && i % this.width === 0) result += '\n';
        let char = '';
        let neighbours = this.getNeighbours(i);

        // if (item === 0) char = '·';
        // if (item === 1) char = 'X';

        if (item === 0) {
          char = '·';
        } else {
          const diffNeighbours = neighbours.filter(n => n !== item).length;
          if (diffNeighbours === 0) {
            char = '☐';
          } else if (diffNeighbours === 1) {
            if (neighbours[0] !== item || neighbours[1] !== item) {
              char = '─';
            } else if (neighbours[2] !== item || neighbours[3] !== item) {
              char = '│';
            }
          } else if (diffNeighbours === 2) {
            if (neighbours[0] !== item && neighbours[2] !== item) {
              char = '╮';
            } else if (neighbours[0] !== item && neighbours[3] !== item) {
              char = '╭';
            } else if (neighbours[1] !== item && neighbours[2] !== item) {
              char = '╯';
            } else if (neighbours[1] !== item && neighbours[3] !== item) {
              char = '╰';
            }
          }
        }

        result += char;
      });
      return result;
    },
  },
  methods: {
    getNeighbours(index) {
      // return {
      //   n: this.getNeighbour(index, 'n'),
      //   s: this.getNeighbour(index, 's'),
      //   e: this.getNeighbour(index, 'e'),
      //   w: this.getNeighbour(index, 'w')
      // }
      return [
        this.getNeighbour(index, 'n'),
        this.getNeighbour(index, 's'),
        this.getNeighbour(index, 'e'),
        this.getNeighbour(index, 'w')
      ]
    },
    getNeighbour(index, direction) {
      if (index >= this.currentData.length || index < 0) {
        console.warn(`getNeighbour index invalid!`);
        return;
      }
      let neighbour;
      switch (direction) {
        case 'n':
          if (index < this.width) {
            neighbour = null;
          } else {
            neighbour = this.currentData[index - this.width];
          }
          break;
        case 's':
          if (index >= this.currentData.length - this.width) {
            neighbour = null;
          } else {
            neighbour = this.currentData[index + this.width];
          }
          break;
        case 'e':
          if (index % this.width === this.width - 1) {
            neighbour = null;
          } else {
            neighbour = this.currentData[index + 1];
          }
          break;
        case 'w':
          if (index % this.width === 0) {
            neighbour = null;
          } else {
            neighbour = this.currentData[index - 1];
          }
          break;
      }
      return neighbour;
    }
  },
  mounted() {
    if (this.startData) this.currentData = this.startData;
  }
}
</script>

<style scoped>
.box-renderer {
  border: 1px solid red;
  width: fit-content;
}

pre {
  font-family: monospace;
  font-size: 40px;
  width: fit-content;
  margin: 0;
}
</style>
