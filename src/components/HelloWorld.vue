<template>
  <div class="hello">
    <svg :width="width" :height="height">
      <g v-for="(rectangle, index) in rectangles" :key="index">
        <rect
          :x="rectangle.x"
          :y="rectangle.y"
          :width="rectangle.dx"
          :height="rectangle.dy"
          style="stroke-width:1;stroke:white;"
        />
        <image
          :x="rectangle.x"
          :y="rectangle.y"
          :width="rectangle.dx"
          :height="rectangle.dy"
          :href="getImage(index)"
          preserveAspectRatio="xMidYMid slice"
        />
        <rect
          :x="(rectangle.x + rectangle.dx / 2)-(rectangle.dx / 2)"
          :y="(rectangle.y + rectangle.dy)-50"
          :width="rectangle.dx"
          :height="50"
          style="stroke-width:1;stroke:white;"
          fill="black"
          fill-opacity="0.5"
        />
        <text
          :x="rectangle.x + rectangle.dx / 2"
          :y="(rectangle.y + rectangle.dy)-30"
          fill="white"
          dominant-baseline="middle"
          text-anchor="middle"
          font-weight="bold"
          style="pointer-events: none;"
        >
        <tspan dy="-0.6em" :style="{ whiteSpace: 'pre-line' }">{{ getFormattedText2(index) }}</tspan>
      
        </text>
        <text
          :x="(rectangle.x + rectangle.dx / 2)"
          :y="(rectangle.y + rectangle.dy )-5"
          fill="white"
          dominant-baseline="middle"
          text-anchor="middle"
          font-weight="bold"
          style="pointer-events: none;"
        >
        {{ getFormattedText(index) }}
        </text>
      </g>
      Sorry, your browser does not support inline SVG.
    </svg>
    <button @click="generateNewPage(1)">page1</button>
    <button @click="generateNewPage(2)">page2</button>
  </div>
</template>

<script>
/* eslint-disable */
export default {
  name: 'HelloWorld',
  props: {
    msg: String
  },
  data() {
    return {
      width: 800,
      height: 600,
      rectangles: [],
      labels: [],
      images: [],
      voteCounts: [],
      datamovies:[]
    };
  },
  mounted() {
    fetch('data.json')
      .then(response => response.json())
      .then(data => {
        this.datamovies =data.results
        this.generateRectangles(data.results,0,10);
      });
  },
  methods: {
    generateNewPage(val){
      console.log(val)
      let indicepage=0
      let lastpage=10
      if(val==1){
        indicepage=0
        lastpage=10
      }else{
        indicepage=10
        lastpage=20
      }
      this.generateRectangles(this.datamovies,indicepage,lastpage);
    },
    padRectangle(rect) {
      if (rect.dx > 2) {
        rect.x += 1;
        rect.dx -= 2;
      }
      if (rect.dy > 2) {
        rect.y += 1;
        rect.dy -= 2;
      }
    },
    layoutRow(sizes, x, y, dx, dy) {
      let coveredArea = sizes.reduce((a, b) => a + b, 0);
      let width = coveredArea / dy;
      let rects = [];
      sizes.forEach(size => {
        rects.push({ x: x, y: y, dx: width, dy: size / width });
        y += size / width;
      });
      return rects;
    },
    layoutCol(sizes, x, y, dx, dy) {
      let coveredArea = sizes.reduce((a, b) => a + b, 0);
      let height = coveredArea / dx;
      let rects = [];
      sizes.forEach(size => {
        rects.push({ x: x, y: y, dx: size / height, dy: height });
        x += size / height;
      });
      return rects;
    },
    layout(sizes, x, y, dx, dy) {
      return dx >= dy ? this.layoutRow(sizes, x, y, dx, dy) : this.layoutCol(sizes, x, y, dx, dy);
    },
    leftoverRow(sizes, x, y, dx, dy) {
      let coveredArea = sizes.reduce((a, b) => a + b, 0);
      let width = coveredArea / dy;
      return { x: x + width, y: y, dx: dx - width, dy: dy };
    },
    leftoverCol(sizes, x, y, dx, dy) {
      let coveredArea = sizes.reduce((a, b) => a + b, 0);
      let height = coveredArea / dx;
      return { x: x, y: y + height, dx: dx, dy: dy - height };
    },
    leftover(sizes, x, y, dx, dy) {
      return dx >= dy ? this.leftoverRow(sizes, x, y, dx, dy) : this.leftoverCol(sizes, x, y, dx, dy);
    },
    worstRatio(sizes, x, y, dx, dy) {
      return Math.max(...this.layout(sizes, x, y, dx, dy).map(rect => Math.max(rect.dx / rect.dy, rect.dy / rect.dx)));
    },
    squarify(sizes, x, y, dx, dy) {
      sizes = sizes.map(size => parseFloat(size));
      if (sizes.length === 0) return [];
      if (sizes.length === 1) return this.layout(sizes, x, y, dx, dy);

      let i = 1;
      while (i < sizes.length && this.worstRatio(sizes.slice(0, i), x, y, dx, dy) >= this.worstRatio(sizes.slice(0, i + 1), x, y, dx, dy)) {
        i++;
      }
      let current = sizes.slice(0, i);
      let remaining = sizes.slice(i);

      let leftoverRect = this.leftover(current, x, y, dx, dy);
      return this.layout(current, x, y, dx, dy).concat(this.squarify(remaining, leftoverRect.x, leftoverRect.y, leftoverRect.dx, leftoverRect.dy));
    },
    normalizeSizes(sizes, dx, dy) {
      let totalSize = sizes.reduce((a, b) => a + b, 0);
      let totalArea = dx * dy;
      return sizes.map(size => size * totalArea / totalSize);
    },
    generateRectangles(data,i,lp) {
      data.sort((a, b) => b.vote_count - a.vote_count);
      const votes = data.slice(i,lp).map(movie => movie.vote_count);
      this.voteCounts = votes;
      this.labels = data.slice(i,lp).map(movie => movie.title);
      this.images = data.slice(i,lp).map(movie => `https://image.tmdb.org/t/p/w500${movie.backdrop_path}`);
      
      const normedSizes = this.normalizeSizes(votes, this.width, this.height);
      const rects = this.squarify(normedSizes, 0, 0, this.width, this.height);
      rects.forEach(this.padRectangle);
      this.rectangles = rects;

      console.log(votes, this.labels, this.images); // eslint-disable-line no-console
    },
    getLabel(index) {
      return this.labels[index] || '';
    },
    getVoteCount(index) {
      return this.voteCounts[index] !== undefined ? `Votes: ${this.voteCounts[index]}` : '';
    },
    getImage(index) {
      return this.images[index] || '';
    },
    getFormattedText(index) {
      const label = this.getLabel(index);
      const voteCount = this.getVoteCount(index);
      const maxWidth = this.rectangles[index].dx - 10; // Adjust margin to fit inside rect
      const words = (voteCount).split(' ');
      let line = '';
      const lines = [];

      words.forEach(word => {
        if (line.length + word.length > maxWidth / 6) { // Approximate character width
          lines.push(line);
          line = word;
        } else {
          line += (line.length ? ' ' : '') + word;
        }
      });

      if (line) {
        lines.push(line);
      }
      console.log(lines.join('\n'))

      return lines.join('\n');
    },
    getFormattedText2(index) {
      const label = this.getLabel(index);
      const voteCount = this.getVoteCount(index);
      const maxWidth = this.rectangles[index].dx - 10; // Adjust margin to fit inside rect
      const words = (label).split(' ');
      let line = '';
      const lines = [];

      words.forEach(word => {
        if (line.length + word.length > maxWidth / 6) { // Approximate character width
          lines.push(line);
          line = word;
        } else {
          line += (line.length ? ' ' : '') + word;
        }
      });

      if (line) {
        lines.push(line);
      }
      console.log(lines.join('\n'))

      return lines.join('\n');
    }
  }
};
</script>

<style scoped>
rect {
  stroke: white;
  stroke-width: 1;
}
text {
  font-size: 10px;
  font-weight: bold;
  text-anchor: middle;
}
</style>
