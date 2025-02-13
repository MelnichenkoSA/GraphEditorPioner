<template>
  <div id="app">
    <svg 
      :width="width" 
      :height="height" 
      @mousedown.prevent="onMouseDown" 
      @mousemove="onMouseMove" 
      @mouseup="onMouseUp" 
      @mouseleave="onMouseUp">
      
      <g>
        <line 
          v-for="x in xGridLines" 
          :key="x" 
          :x1="x" :y1="0" 
          :x2="x" :y2="height" 
          stroke="#ddd"/>
        <line 
          v-for="y in yGridLines" 
          :key="y" 
          :x1="0" :y1="y" 
          :x2="width" :y2="y" 
          stroke="#ddd"/>
      </g>

      
      <line :x1="0" :y1="Math.floor(height / 2 / gridStep) * gridStep" :x2="width" :y2="Math.floor(height / 2 / gridStep) * gridStep" stroke="black" />
      <line :x1="Math.floor(width / 2 / gridStep) * gridStep" :y1="0" :x2="Math.floor(width / 2 / gridStep) * gridStep" :y2="height" stroke="black" />

      
      <text :x="Math.floor(width / 2 / gridStep) * gridStep + 5" :y="15" fill="black">Y</text>
      <text :x="width - 15" :y="Math.floor(height / 2 / gridStep) * gridStep - 5" fill="black" text-anchor="end">X</text>

      
      <polyline 
        :points="linePoints" 
        fill="none" 
        stroke="blue"/>

      
      <line 
        v-if="points.length > 1 && connectFirstLast" 
        :x1="points[0].x + width / 2" 
        :y1="height / 2 - points[0].y" 
        :x2="points[points.length - 1].x + width / 2" 
        :y2="height / 2 - points[points.length - 1].y" 
        stroke="green" 
        stroke-dasharray="5,5"/>

      
      <circle 
        v-for="(point, index) in points" 
        :key="index" 
        :cx="point.x + width / 2" 
        :cy="height / 2 - point.y" 
        r="5" 
        fill="red" 
        @mousedown.prevent="startDrag(index)"/>
    </svg>
    <div style="display: flex; flex-direction: column; gap: 15px; max-width: 200px; margin: 0 auto;">
      <button style="width: 100%;" @click="toggleIntegerMode">{{ integerMode ? 'Отключить целочисленный режим' : 'Включить целочисленный режим' }}</button>
      <button style="width: 100%;" @click="toggleConnectFirstLast">{{ connectFirstLast ? 'Разъединить первую и последнюю точку' : 'Соединить первую и последнюю точку' }}</button>
      <label>
        Шаг сетки:
        <input type="number" v-model.number="gridStep" min="1" @input="updateGridStep" />
      </label>
      <label>
        Масштаб:
        <input type="number" v-model.number="scale" step="0.1" min="0.1" />
      </label>
      <div>
        <input type="file" @change="onFileChange" />
      </div>
      <div>
        <label>
          X:
          <input type="number" v-model.number="newPointX" />
        </label>
        <label>
          Y:
          <input type="number" v-model.number="newPointY" />
        </label>
        <button style="width: 100%;" @click="addPoint">Добавить точку</button>
        <button style="width: 100%;" @click="exportData">Экспорт данных</button>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      width: 400,
      height: 300,
      points: [],
      draggingPointIndex: null,
      isShiftPressed: false,
      isCtrlPressed: false,
      integerMode: false,
      gridStep: 10,
      scale: 1,
      connectFirstLast: false,
      newPointX: 0,
      newPointY: 0,
    };
  },
  computed: {
    xGridLines() {
      return Array.from({ length: Math.ceil(this.width / this.gridStep) + 1 }, (_, i) => i * this.gridStep);
    },
    yGridLines() {
      return Array.from({ length: Math.ceil(this.height / this.gridStep) + 1 }, (_, i) => i * this.gridStep);
    },
    linePoints() {
      return this.points.map((p) => `${p.x + this.width / 2},${this.height / 2 - p.y}`).join(" ");
    },
  },
  methods: {
    startDrag(index) {
      this.draggingPointIndex = index;
    },
    onMouseMove(event) {
      if (this.draggingPointIndex === null) return;

      const rect = event.currentTarget.getBoundingClientRect();
      let x = (event.clientX - rect.left) / this.scale - this.width / 2;
      let y = this.height / 2 - (event.clientY - rect.top) / this.scale;

      if (this.integerMode) {
        x = Math.round(x / this.gridStep) * this.gridStep;
        y = Math.round(y / this.gridStep) * this.gridStep;
      }

      if (this.isShiftPressed) {
        x = this.points[this.draggingPointIndex].x;
      }

      if (this.isCtrlPressed) {
        y = this.points[this.draggingPointIndex].y;
      }

      this.points.splice(this.draggingPointIndex, 1, { x, y });
    },
    onMouseDown(event) {
      this.isShiftPressed = event.shiftKey;
      this.isCtrlPressed = event.ctrlKey;
    },
    onMouseUp() {
      this.draggingPointIndex = null;
      this.isShiftPressed = false;
      this.isCtrlPressed = false;
    },
    toggleIntegerMode() {
      this.integerMode = !this.integerMode;
    },
    toggleConnectFirstLast() {
      this.connectFirstLast = !this.connectFirstLast;
    },
    addPoint() {
      const x = this.newPointX * 10;
      const y = this.newPointY * 10;
      this.points.push({ x, y });
    },
    updateGridStep() {
      this.points = this.points.map((point) => ({
        x: Math.round(point.x / this.gridStep) * this.gridStep,
        y: Math.round(point.y / this.gridStep) * this.gridStep,
      }));
    },
    onFileChange(event) {
      const file = event.target.files[0];
      if (!file) return;

      const reader = new FileReader();
      reader.onload = (e) => {
        const lines = e.target.result.split("\n");
        const newPoints = [];

        for (let i = 4; i <= 7; i++) { 
          const line = lines[i]?.trim();
          console.log(`Processing line ${i + 1}:`, line); 
          const columns = line?.split(/\s+/);
          if (columns && columns.length >= 5) { 
            const xStr = columns[3];
            const yStr = columns[4];
            const x = parseFloat(xStr) * 10; 
            const y = parseFloat(yStr) * 10; 
            console.log(`Parsed point: x=${x}, y=${y}`); 
            if (!isNaN(x) && !isNaN(y)) {
              newPoints.push({ x, y });
            } else {
              console.error(`Failed to parse point at line ${i + 1}: x=${xStr}, y=${yStr}`);
            }
          } else {
            console.error(`Line ${i + 1} does not have enough columns:`, columns);
          }
        }

        this.points = newPoints;
        console.log(`Loaded points:`, this.points); 
      };
      reader.readAsText(file);
    },
    exportData() {
    
    const lines = Array(8).fill("");

    
    this.points.forEach((point, index) => {
      if (index < 4) { 
        const pointIndex = (index + 1).toString(); 
        const x = (point.x / 10).toFixed(1); 
        const y = (point.y / 10).toFixed(1); 

        
        let line = " ".repeat(9) + pointIndex.padEnd(2, " ");
        line = line.padEnd(14, " ") + "0"; 
        line = line.padEnd(19, " ") + "0"; 
        line = line.padEnd(37, " "); 
        line += x.padEnd(20, " "); 
        line += y.padEnd(20, " "); 
        lines[4 + index] = line; 
      }
    });
    const content = lines.join("\n");
    const blob = new Blob([content], { type: "text/plain" });
    const url = URL.createObjectURL(blob);

    const link = document.createElement("a");
    link.href = url;
    link.download = "exported_data.txt";
    document.body.appendChild(link);
    link.click();

    document.body.removeChild(link);
    URL.revokeObjectURL(url);
  }
  },
};
</script>

<style>
svg {
  user-select: none;
  border: 1px solid black;
  margin-bottom: 10px;
}
</style>











































