<template>
  <div>
    <!-- 當前節點 -->
    <div 
      :class="['meter-row', { 
        selected: meter.selected, 
        expanded: isExpanded(meter.id),
        'drop-target': dropTargetMeter?.id === meter.id 
      }]"
      :style="{ marginLeft: depth * 40 + 'px' }"
      draggable="true"
      @dragstart="handleDragStart($event, meter, depth)"
      @dragend="handleDragEnd"
      @dragover.prevent="handleDragOver"
      @dragenter.prevent="handleDragEnter($event, meter)"
      @dragleave.prevent="handleDragLeave"
      @drop.prevent.stop="handleDropOnMeter($event, meter)"
    >
      <div class="meter-select" @click.stop>
        <input 
          type="checkbox" 
          :checked="meter.selected" 
          @click.stop="toggleSelect($event, meter, depth)"
        >
      </div>
      <div class="meter-drag-handle">
        <span class="drag-icon">⋮⋮</span>
      </div>
      <div class="meter-content" @click="handleRowClick(meter)">
        <div class="meter-name">
          <span v-if="getChildNodes(meter.id).length > 0" class="expand-icon">
            {{ isExpanded(meter.id) ? '▼' : '▶' }}
          </span>
          {{ meter.name }}
        </div>
      </div>
      <div class="meter-id">ID: {{ meter.id }}</div>
    </div>

    <!-- 遞迴渲染子節點 -->
    <template v-if="isExpanded(meter.id)">
      <MeterRow 
        v-for="childMeter in getChildNodes(meter.id)" 
        :key="childMeter.id"
        :meter="childMeter" 
        :depth="depth + 1"
        :isExpanded="isExpanded"
        :getChildNodes="getChildNodes"
        :handleDragStart="handleDragStart"
        :handleDragEnd="handleDragEnd"
        :handleDragOver="handleDragOver"
        :handleDragEnter="handleDragEnter"
        :handleDragLeave="handleDragLeave"
        :handleDropOnMeter="handleDropOnMeter"
        :toggleSelect="toggleSelect"
        :handleRowClick="handleRowClick"
        :dropTargetMeter="dropTargetMeter"
      />
    </template>
  </div>
</template>

<script setup>
defineProps({
  meter: {
    type: Object,
    required: true
  },
  depth: {
    type: Number,
    required: true
  },
  isExpanded: {
    type: Function,
    required: true
  },
  getChildNodes: {
    type: Function,
    required: true
  },
  handleDragStart: {
    type: Function,
    required: true
  },
  handleDragEnd: {
    type: Function,
    required: true
  },
  handleDragOver: {
    type: Function,
    required: true
  },
  handleDragEnter: {
    type: Function,
    required: true
  },
  handleDragLeave: {
    type: Function,
    required: true
  },
  handleDropOnMeter: {
    type: Function,
    required: true
  },
  toggleSelect: {
    type: Function,
    required: true
  },
  handleRowClick: {
    type: Function,
    required: true
  },
  dropTargetMeter: {
    type: Object,
    default: null
  }
})
</script>

<style scoped>
.meter-row {
  display: flex;
  align-items: center;
  padding: 18px 20px;
  background: #f8f9fa;
  border: 2px solid #e0e0e0;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.2s ease;
  position: relative;
  min-height: 60px;
}

.meter-row:hover {
  border-color: #3498db;
  background: #ecf7fd;
  transform: translateX(5px);
}

.meter-row.selected {
  border-color: #5CADAD;
  background: #E8F5F5;
  box-shadow: 0 2px 8px rgba(92, 173, 173, 0.2);
}

.meter-row.expanded {
  background: #f0f8ff;
}

.meter-row.drop-target {
  background: #fff9e6;
  border-color: #ffa500;
  border-width: 3px;
  border-style: dashed;
  box-shadow: 0 0 15px rgba(255, 165, 0, 0.4);
  transform: scale(1.02);
  position: relative;
}

.meter-row.drop-target::after {
  content: '放開以移動到此處';
  position: absolute;
  right: 20px;
  top: 50%;
  transform: translateY(-50%);
  background: #ffa500;
  color: white;
  padding: 4px 12px;
  border-radius: 4px;
  font-size: 12px;
  font-weight: 600;
  pointer-events: none;
  white-space: nowrap;
}

.expand-icon {
  display: inline-block;
  width: 20px;
  color: #3498db;
  font-size: 12px;
  margin-right: 5px;
}

.meter-select {
  display: flex;
  align-items: center;
  margin-right: 15px;
}

.meter-select input[type="checkbox"] {
  width: 18px;
  height: 18px;
  cursor: pointer;
}

.meter-drag-handle {
  display: flex;
  align-items: center;
  margin-right: 15px;
  color: #95a5a6;
  cursor: move;
  user-select: none;
}

.drag-icon {
  font-size: 18px;
  line-height: 1;
}

.meter-content {
  flex: 1;
  display: flex;
  gap: 20px;
  align-items: center;
  cursor: pointer;
}

.meter-name {
  font-weight: 600;
  color: #2c3e50;
  font-size: 16px;
  min-width: 150px;
}

.meter-id {
  font-size: 13px;
  color: #95a5a6;
  min-width: 80px;
  text-align: right;
}

/* 拖拽效果 */
.meter-row[draggable="true"] {
  cursor: grab;
}

.meter-row[draggable="true"]:active {
  cursor: grabbing;
}

.meter-row.dragging {
  opacity: 0.4;
  transform: scale(0.95);
}
</style>
