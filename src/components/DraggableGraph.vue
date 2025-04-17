<template>
  <div id="app">
    <div class="container">
      <!-- Кнопки для графика (слева) -->
      <div class="graph-controls">
        <button @click="toggleIntegerMode">{{ integerMode ? 'Отключить целочисленный режим' : 'Включить целочисленный режим' }}</button>
        <button @click="toggleConnectFirstLast">{{ connectFirstLast ? 'Разъединить первую и последнюю точку' : 'Соединить первую и последнюю точку' }}</button>
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
          <button @click="addPoint">Добавить точку</button>
          <button @click="exportData">Экспорт данных</button>
        </div>
        <label>
          <input type="checkbox" v-model="showIndexes" />
          Показывать индексы точек
        </label>
      </div>

      
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

        <polygon 
        v-if="points.length > 1 && connectFirstLast"
        :points="polygonPoints"
        fill="rgba(0, 0, 255, 0.3)"
        stroke="none"
      />
      
      <polyline 
        :points="linePoints" 
        fill="none" 
        stroke="blue"
        stroke-width="2"
      />

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
          :fill="getPointColor(point)"
          @contextmenu.prevent.stop="openContextMenu($event, index)"
          @mousedown.prevent="startDrag(index)"/>

        <g v-if="showIndexes" style="pointer-events: none;">
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

      
      <div class="editor-section">
        
        <div v-if="isEditorVisible" class="file-editor">
          <h3>Импортированный файл</h3>
          <textarea v-model="importedFileContent" rows="20" cols="40"></textarea>

         
          <div class="replace-section">
            <label>
              Ключ:
              <input type="text" v-model="replaceKey" />
            </label>
            <label>
              Значение:
              <input type="text" v-model="replaceValue" />
            </label>
            <button @click="replaceText">Изменить</button>
          </div>
        </div>

        
        <button v-if="importedFileContent !== null && !isEditorVisible" @click="openFileEditor">
          Открыть редактор
        </button>

        
        <button v-if="isEditorVisible" @click="closeFileEditor">
          Закрыть редактор
        </button>
      </div>
    </div>

    
    <ul v-if="contextMenu.visible" 
      class="context-menu" 
      :style="{ top: contextMenu.y + 'px', left: contextMenu.x + 'px' }">
      <li v-if="contextMenu.targetIndex === null" @click="addPointAtCursor">Добавить точку</li>
      <li v-if="contextMenu.targetIndex !== null" @click="removePoint(contextMenu.targetIndex)">Удалить точку</li>
      <li v-if="contextMenu.targetIndex === null" @click="clearPoints">Удалить все точки</li>
      <li v-if="contextMenu.targetIndex !== null" @click="toggleConstraint(contextMenu.targetIndex, 'x')">
        Зафиксировать X
      </li>
      <li v-if="contextMenu.targetIndex !== null" @click="toggleConstraint(contextMenu.targetIndex, 'y')">
        Зафиксировать Y
      </li>
      <li v-if="contextMenu.targetIndex !== null" @click="toggleConstraint(contextMenu.targetIndex, 'xy')">
        Зафиксировать X и Y
      </li>
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
      importedFileContent: null, 
      isEditorVisible: false,
      replaceKey: "", 
      replaceValue: "", 
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
    polygonPoints() {
    if (!this.points.length) return '';
    
    // Создаем массив точек для полигона
    const points = this.points.map(p => `${p.x + this.width / 2},${this.height / 2 - p.y}`);
    
    // Если нужно соединить первую и последнюю точку, добавляем первую точку в конец
    if (this.connectFirstLast && this.points.length > 2) {
      const first = this.points[0];
      points.push(`${first.x + this.width / 2},${this.height / 2 - first.y}`);
    }
    
    return points.join(' ');
  }
  },
  methods: {
    getPointColor(point) {
      if (!point.constraints) {
        return "green";
      }
      if (point.constraints.x === 1 && point.constraints.y === 1) {
        return "purple";
      }
      if (point.constraints.x === 1) {
        return "blue";
      }
      if (point.constraints.y === 1) {
        return "red";
      }
      return "green";
    },
    openContextMenu(event, index = null) {
      event.preventDefault();
      event.stopPropagation();

      this.contextMenu.visible = true;
      this.contextMenu.x = event.clientX;
      this.contextMenu.y = event.clientY;
      this.contextMenu.targetIndex = index;

      if (index === null) {
        const svgRect = event.currentTarget.getBoundingClientRect();
        this.contextMenu.cursorX = (event.clientX - svgRect.left) - this.width / 2;
        this.contextMenu.cursorY = this.height / 2 - (event.clientY - svgRect.top);
      }
    },
    toggleConstraint(index, type) {
      if (index === null || index >= this.points.length) return;
      const point = this.points[index];

      if (!point.constraints) {
        point.constraints = { x: 0, y: 0 };
      }

      if (type === "x") {
        point.constraints.x = point.constraints.x === 1 ? 0 : 1;
      } else if (type === "y") {
        point.constraints.y = point.constraints.y === 1 ? 0 : 1;
      } else if (type === "xy") {
        if (point.constraints.x === 1 && point.constraints.y === 1) {
          point.constraints.x = 0;
          point.constraints.y = 0;
        } else {
          point.constraints.x = 1;
          point.constraints.y = 1;
        }
      }

      this.contextMenu.visible = false;
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

      const point = this.points[this.draggingPointIndex];

      if (this.integerMode) {
        x = Math.round(x / this.gridStep) * this.gridStep;
        y = Math.round(y / this.gridStep) * this.gridStep;
      }

      if (point.constraints?.x === 1) {
        x = point.x;
      }
      if (point.constraints?.y === 1) {
        y = point.y;
      }

      this.points.splice(this.draggingPointIndex, 1, { ...point, x, y });
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

      const newIndex = this.points.length > 0
        ? Math.max(...this.points.map(p => p.index ?? 0)) + 1
        : 1;

      this.points.push({ x, y, index: newIndex, constraints: { x: 0, y: 0 } });
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
        const content = e.target.result;
        this.importedFileContent = content; 
        this.isEditorVisible = true; 
        this.parseFileContent(content); 
      };
      reader.readAsText(file);
    },

    parseFileContent(content) {
      const lines = content.split("\n").map(line => line.trim());
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
          const constraintX = isNaN(parseInt(columns[1])) ? 0 : parseInt(columns[1]);
          const constraintY = isNaN(parseInt(columns[2])) ? 0 : parseInt(columns[2]);
          pointMap[index] = { x, y, index, constraints: { x: constraintX, y: constraintY } };

          if (!isNaN(index) && !isNaN(x) && !isNaN(y)) {
            if (pointMap[index]) {
              console.warn(`Дубликат точки с индексом ${index}, перезапись!`, pointMap[index]);
            }
            pointMap[index] = { x, y, index, constraints: { x: constraintX, y: constraintY } };
          } else {
            console.error(`Ошибка: некорректные данные в строке ${i + 1}`, columns);
          }
        } else {
          console.error(`Ошибка: недостаточно данных в строке ${i + 1}`, columns);
        }
      }

      console.log("Считанные точки (до сортировки):", pointMap);

      lastLine.forEach(index => {
        if (pointMap[index]) {
          newPoints.push(pointMap[index]);
        } else {
          console.warn(`Точка с индексом ${index} не найдена`);
        }
      });

      this.points = newPoints;
      console.log("🔹 Загруженные точки (с индексами и правильным порядком):", this.points);
    },
openFileEditor() {
      this.isEditorVisible = true; 
    },
closeFileEditor() {
      
      this.isEditorVisible = false;
    },
    replaceText() {
      if (!this.replaceKey) {
        alert("Введите ключ для замены!");
        return;
      }

      
      this.importedFileContent = this.importedFileContent.replace(
        new RegExp(this.replaceKey, "g"),
        this.replaceValue
      );

      
      this.parseFileContent(this.importedFileContent);
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
            line = line.padEnd(14, " ") + constraintX.toString(); 
            line = line.padEnd(19, " ") + constraintY.toString(); 
            line = line.padEnd(37, " "); 
            line += x.padEnd(20, " "); 
            line += y.padEnd(20, " "); 
            

            lines[4 + index] = line; 
        }
    });

    lines.push("1 1");

    let pointsCopy = [...this.points];

    pointsCopy.sort((a, b) => b.y - a.y && b.x - a.x);
    let maxYMaxX = pointsCopy.shift();

    pointsCopy.sort((a, b) => b.y - a.y);
    let maxYMinX = pointsCopy.shift();

    pointsCopy.sort((a, b) => a.x - b.x);
    let minYMinX = pointsCopy.shift();

    let minYMaxX = pointsCopy[0]; 

    const sortedPoints = [maxYMaxX, maxYMinX, minYMinX, minYMaxX];

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
.container {
  display: flex;
  gap: 20px; 
  align-items: flex-start; 
  padding: 20px; 
}

.graph-controls {
  display: flex;
  flex-direction: column;
  gap: 15px; 
  max-width: 200px; 
}

svg {
  user-select: none;
  border: 1px solid black;
  margin-bottom: 10px;
}

.editor-section {
  display: flex;
  flex-direction: column;
  gap: 15px; 
}

.file-editor {
  background: white;
  padding: 20px;
  border: 1px solid #ccc;
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.2);
  width: 400px; 
  max-height: 80vh; 
  overflow-y: auto; 
}

.file-editor textarea {
  width: 100%;
  margin-bottom: 10px;
}

.file-editor button {
  padding: 5px 10px;
  background: #f0f0f0;
  border: 1px solid #ccc;
  cursor: pointer;
}

.file-editor button:hover {
  background: #ddd;
}

.replace-section {
  margin-top: 10px;
}

.replace-section label {
  display: block;
  margin-bottom: 10px;
}

.replace-section input {
  width: 100%;
  padding: 5px;
  margin-top: 5px;
}

button {
  padding: 5px 10px;
  background: #f0f0f0;
  border: 1px solid #ccc;
  cursor: pointer;
}

button:hover {
  background: #ddd;
}

label {
  display: block;
  margin-bottom: 10px;
}

input[type="number"], input[type="text"] {
  width: 100%;
  padding: 5px;
  margin-top: 5px;
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
polygon {
  pointer-events: none; /* Чтобы не мешало взаимодействию с точками */
}

polyline {
  pointer-events: none;
  transition: fill-opacity 0.3s ease;
}
</style>