<template>
  <div id="app">
    <svg 
      :width="width" 
      :height="height" 
      @click="closeContextMenu"
      @contextmenu.prevent="openContextMenu"
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
        @contextmenu.prevent.stop="openContextMenu($event, index)"
        @mousedown.prevent="startDrag(index)"/>

        <g v-if="showIndexes">
        <text v-for="(point, index) in points" 
        :key="'text-' + index"
        :x="point.x + width / 2 + 8" 
        :y="height / 2 - point.y - 8"
        font-size="12" 
        fill="black">
        {{ point.index }}
        </text>
        </g>

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
    <label>
      <input type="checkbox" v-model="showIndexes" />
      Показывать индексы точек
    </label>

    <ul v-if="contextMenu.visible" 
        class="context-menu" 
        :style="{ top: contextMenu.y + 'px', left: contextMenu.x + 'px' }">
      <li v-if="contextMenu.targetIndex === null" @click="addPointAtCursor">Добавить точку</li>
      <li v-if="contextMenu.targetIndex !== null" @click="removePoint(contextMenu.targetIndex)">Удалить точку</li>
      <li v-if="contextMenu.targetIndex === null" @click="clearPoints">Удалить все точки</li>
    </ul>
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
      showIndexes: true,
      contextMenu: {
        visible: false,
        x: 0,
        y: 0,
        targetIndex: null, 
        cursorX: 0,
        cursorY: 0,
      },
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
    openContextMenu(event, index = null) {
    event.preventDefault();
    event.stopPropagation(); 

    console.log(`Контекстное меню: X=${event.clientX}, Y=${event.clientY}, targetIndex=${index}`);

    this.contextMenu.visible = true;
    this.contextMenu.x = event.clientX;
    this.contextMenu.y = event.clientY;
    this.contextMenu.targetIndex = index;

    if (index === null) {
        const svgRect = event.currentTarget.getBoundingClientRect(); 
        this.contextMenu.cursorX = (event.clientX - svgRect.left) - this.width / 2;
        this.contextMenu.cursorY = this.height / 2 - (event.clientY - svgRect.top);
        
        console.log(`Добавление точки: X=${this.contextMenu.cursorX}, Y=${this.contextMenu.cursorY}`);
    }
    },
    onContextMenu(point) {
        const options = [
            { text: "Зафиксировать X", action: () => this.toggleConstraint(point, "x") },
            { text: "Зафиксировать Y", action: () => this.toggleConstraint(point, "y") },
            { text: "Зафиксировать X и Y", action: () => this.toggleConstraint(point, "xy") },
        ];
        this.showContextMenu(options, point);
    },

    toggleConstraint(point, type) {
        if (!point.constraints) {
            point.constraints = { x: 0, y: 0 };
        }
        if (type === "x") {
            point.constraints.x = point.constraints.x === 1 ? 0 : 1;
        } else if (type === "y") {
            point.constraints.y = point.constraints.y === 1 ? 0 : 1;
        } else if (type === "xy") {
            point.constraints.x = point.constraints.x === 1 && point.constraints.y === 1 ? 0 : 1;
            point.constraints.y = point.constraints.y === 1 && point.constraints.x === 1 ? 0 : 1;
        }
        this.$forceUpdate(); 
    },
    closeContextMenu() {
      this.contextMenu.visible = false;
    },
    addPointAtCursor() {
    const newIndex = this.points.length > 0 
        ? Math.max(...this.points.map(p => p.index ?? 0)) + 1 
        : 1;

    this.points.push({
        x: this.contextMenu.cursorX,
        y: this.contextMenu.cursorY,
        index: newIndex
    });

    console.log(`Добавлена точка: X=${this.contextMenu.cursorX}, Y=${this.contextMenu.cursorY}, Index=${newIndex}`);
    this.closeContextMenu();
    },
    removePoint(index) {
      this.points.splice(index, 1);
      this.closeContextMenu();
    },
    clearPoints() {
      this.points = [];
      this.closeContextMenu();
    },
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

    const movedPoint = { 
        x, 
        y, 
        index: this.points[this.draggingPointIndex].index ?? (this.draggingPointIndex + 1) 
    };

    this.points.splice(this.draggingPointIndex, 1, movedPoint);
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
        const lines = e.target.result.split("\n").map(line => line.trim());
        const newPoints = [];

        const lastLine = lines[lines.length - 1]?.split(/\s+/).map(n => parseInt(n));
        if (!lastLine || lastLine.length < 4 || lastLine.some(isNaN)) {
            console.error("Ошибка: некорректная последняя строка", lastLine);
            return;
        }

        console.log("Порядок точек (из последней строки):", lastLine);

        const pointMap = {};
        for (let i = 4; i <= 7; i++) {
            const columns = lines[i]?.split(/\s+/).filter(Boolean);
            if (columns.length >= 5) { 
                const index = parseInt(columns[0]); 
                const x = parseFloat(columns[3]) * 10;
                const y = parseFloat(columns[4]) * 10;
                const constraintX = isNaN(parseInt(columns[14])) ? 0 : parseInt(columns[14]);
                const constraintY = isNaN(parseInt(columns[19])) ? 0 : parseInt(columns[19]);

                if (!isNaN(index) && !isNaN(x) && !isNaN(y)) {
                    if (pointMap[index]) {
                        console.warn(`⚠ Дубликат точки с индексом ${index}, перезапись!`, pointMap[index]);
                    }
                    pointMap[index] = { x, y, index, constraints: { x: constraintX, y: constraintY } };
                } else {
                    console.error(`❌ Ошибка: некорректные данные в строке ${i + 1}`, columns);
                }
            } else {
                console.error(`❌ Ошибка: недостаточно данных в строке ${i + 1}`, columns);
            }
        }

        console.log("Считанные точки (до сортировки):", pointMap);

        lastLine.forEach(index => {
            if (pointMap[index]) {
                newPoints.push(pointMap[index]);
            } else {
                console.warn(`⚠ Точка с индексом ${index} не найдена`);
            }
        });

        this.points = newPoints;
        console.log("🔹 Загруженные точки (с индексами и правильным порядком):", this.points);
    };

    reader.readAsText(file);
},
exportData() {
    if (!this.points || this.points.length < 4) {
        console.error("Ошибка: недостаточно точек для экспорта", this.points);
        return;
    }

    const lines = Array(8).fill("");

    this.points.forEach((point, index) => {
        if (index < 4) { 
            if (point.index === undefined) {
                console.error(`Ошибка: точка ${index} не имеет index`, point);
                return;
            }

            const pointIndex = point.index.toString();
            const x = (point.x / 10).toFixed(1);
            const y = (point.y / 10).toFixed(1);
            const constraintX = point.constraints?.x || 0; 
            const constraintY = point.constraints?.y || 0; 

            let line = " ".repeat(9) + pointIndex.padEnd(5, " "); 
            line = line.padEnd(14, " ") + "0"; 
            line = line.padEnd(19, " ") + "0"; 
            line = line.padEnd(37, " "); 
            line += x.padEnd(20, " "); 
            line += y.padEnd(20, " "); 
            line += constraintX.toString().padEnd(5, " "); // 📌 15-й столбец
            line += constraintY.toString().padEnd(5, " "); // 📌 20-й столбец 

            lines[4 + index] = line; 
        }
    });

    lines.push("1 1");

    // 📌 Определяем 4 ключевые точки
        // Копируем массив, чтобы не мутировать оригинал
    let pointsCopy = [...this.points];

    // 1️⃣ Находим самую правую верхнюю точку (max Y, max X)
    pointsCopy.sort((a, b) => b.y - a.y && b.x - a.x);
    let maxYMaxX = pointsCopy.shift();

    // 2️⃣ Находим самую левую верхнюю точку (max Y, min X)
    pointsCopy.sort((a, b) => b.y - a.y);
    let maxYMinX = pointsCopy.shift();

    // 3️⃣ Находим самую левую нижнюю точку (min Y, min X)
    pointsCopy.sort((a, b) => a.x - b.x);
    let minYMinX = pointsCopy.shift();

    // 4️⃣ Находим самую правую нижнюю точку (min Y, max X)
    let minYMaxX = pointsCopy[0]; // Осталась последняя точка

    const sortedPoints = [maxYMaxX, maxYMinX, minYMinX, minYMaxX];


    // 📌 Формируем строку с индексами
    const pointIndices = sortedPoints.map(point => point.index.toString()).join("    ");
    lines.push(pointIndices);

    console.log("✅ Правильный порядок точек:", pointIndices);

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
.context-menu {
  position: absolute;
  background: white;
  border: 1px solid #ccc;
  padding: 5px 0;
  list-style: none;
  box-shadow: 2px 2px 10px rgba(0,0,0,0.2);
  z-index: 1000;
}
.context-menu li {
  padding: 5px 15px;
  cursor: pointer;
}
.context-menu li:hover {
  background: #eee;
}
</style>