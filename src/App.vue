<script setup>
import { ref, computed, onMounted } from 'vue'
import MeterRow from './components/MeterRow.vue'

// 電表數據，按層級組織
const layers = ref({
  0: [],
  1: [],
  2: []
})

// 從 API 獲取節點資料
const fetchNodes = async () => {
  try {
    const response = await fetch('http://localhost:3001/nodes')
    if (!response.ok) {
      throw new Error('無法獲取節點資料')
    }
    const nodes = await response.json()
    
    console.log('API 返回的原始資料:', nodes)
    
    // 將節點按層級分類（根據 parent_id 關係）
    layers.value = {
      0: [],
      1: [],
      2: []
    }
    
    // 建立 ID 到節點的映射
    const nodeMap = new Map()
    nodes.forEach(node => {
      nodeMap.set(node.id, {
        ...node,
        selected: false,
        layer: -1 // 初始未計算
      })
    })
    
    console.log('節點映射表:', Array.from(nodeMap.entries()))
    
    // 遞迴計算節點層級
    const calculateLayer = (nodeId, visited = new Set()) => {
      // 防止循環引用
      if (visited.has(nodeId)) {
        console.warn('檢測到循環引用:', nodeId)
        return 0
      }
      visited.add(nodeId)
      
      const node = nodeMap.get(nodeId)
      if (!node) {
        console.warn('找不到節點 ID:', nodeId)
        return 0
      }
      
      // 如果已經計算過，直接返回
      if (node.layer !== -1) {
        return node.layer
      }
      
      // 如果沒有父節點，為第0層
      if (node.parent_id === null || node.parent_id === undefined) {
        node.layer = 0
        console.log(`節點 ID:${node.id} "${node.name}" - 根節點 → 第0層`)
        return 0
      }
      
      // 有父節點，遞迴計算父節點的層級
      const parentNode = nodeMap.get(node.parent_id)
      if (!parentNode) {
        console.warn(`節點 ID:${node.id} 的父節點 ID:${node.parent_id} 不存在`)
        node.layer = 0
        return 0
      }
      
      const parentLayer = calculateLayer(node.parent_id, visited)
      node.layer = parentLayer + 1
      console.log(`節點 ID:${node.id} "${node.name}" - 父節點 ID:${node.parent_id} "${parentNode.name}" (第${parentLayer}層) → 第${node.layer}層`)
      return node.layer
    }
    
    // 計算所有節點的層級
    nodes.forEach(node => {
      calculateLayer(node.id)
    })
    
    // 將節點分配到對應層級
    nodeMap.forEach(node => {
      if (node.layer >= 0 && node.layer <= 2) {
        layers.value[node.layer].push(node)
      } else if (node.layer > 2) {
        // 超過第2層的節點放在第2層
        node.layer = 2
        layers.value[2].push(node)
      }
    })
    
    console.log('最終層級分配:')
    console.log('第0層:', layers.value[0].map(n => `ID:${n.id} ${n.name}`))
    console.log('第1層:', layers.value[1].map(n => `ID:${n.id} ${n.name} (parent:${n.parent_id})`))
    console.log('第2層:', layers.value[2].map(n => `ID:${n.id} ${n.name} (parent:${n.parent_id})`))
  } catch (error) {
    console.error('獲取節點資料失敗:', error)
    alert('無法連接到伺服器，請確認 API 服務是否運行')
  }
}

// 組件掛載時獲取資料
onMounted(() => {
  fetchNodes()
})

// 計算已選擇的電表數量
const selectedCount = computed(() => {
  let count = 0
  Object.values(layers.value).forEach(layerMeters => {
    count += layerMeters.filter(m => m.selected).length
  })
  return count
})

// 當前選擇的層級
const activeLayer = ref(null)

// 當前展開的節點ID集合（用於顯示其子節點）
const expandedNodeIds = ref(new Set())

// 控制各層級的展開狀態
const expandedLayers = ref({
  0: true,  // 第0層永遠顯示
  1: true,  // 暫時設為 true 以便查看資料
  2: true   // 暫時設為 true 以便查看資料
})

// 拖拽相關
const draggedMeter = ref(null)
const draggedFromLayer = ref(null)
const dropTargetMeter = ref(null)

// 根據父節點ID篩選子節點
const getChildNodes = (parentId) => {
  let children = []
  Object.values(layers.value).forEach(layerNodes => {
    children = children.concat(layerNodes.filter(node => node.parent_id === parentId))
  })
  return children
}

// 檢查節點是否已展開
const isExpanded = (meterId) => {
  return expandedNodeIds.value.has(meterId)
}

// 點擊電表行展開子層
const handleRowClick = (meter) => {
  // 切換展開狀態
  if (expandedNodeIds.value.has(meter.id)) {
    expandedNodeIds.value.delete(meter.id) // 收起
  } else {
    expandedNodeIds.value.add(meter.id) // 展開
  }
  // 觸發響應式更新
  expandedNodeIds.value = new Set(expandedNodeIds.value)
}

// 切換電表選擇狀態
const toggleSelect = (event, meter, layerNum) => {
  // 如果沒有任何選擇，直接選擇
  if (activeLayer.value === null) {
    meter.selected = true
    activeLayer.value = layerNum
  }
  // 如果選擇的是同一層級
  else if (activeLayer.value === layerNum) {
    meter.selected = !meter.selected
    // 檢查該層級是否還有選中的電表
    const hasSelected = layers.value[layerNum].some(m => m.selected)
    if (!hasSelected) {
      activeLayer.value = null
    }
  }
  // 如果嘗試選擇不同層級，阻止預設行為並提示錯誤
  else {
    event.preventDefault()
    alert(`只能在同一層級內選擇電表。目前已選擇第 ${activeLayer.value} 層的電表，請先取消選擇後再選擇其他層級。`)
  }
}

// 開始拖拽
const handleDragStart = (event, meter, layerNum) => {
  draggedMeter.value = meter
  draggedFromLayer.value = layerNum
  event.dataTransfer.effectAllowed = 'move'
  event.target.classList.add('dragging')
}

// 拖拽結束
const handleDragEnd = (event) => {
  event.target.classList.remove('dragging')
  draggedMeter.value = null
  draggedFromLayer.value = null
  dropTargetMeter.value = null
}

// 拖拽經過
const handleDragOver = (event) => {
  event.preventDefault()
  event.dataTransfer.dropEffect = 'move'
}

// 拖拽進入節點
const handleDragEnter = (event, meter) => {
  event.preventDefault()
  event.stopPropagation()
  if (draggedMeter.value && draggedMeter.value.id !== meter.id) {
    dropTargetMeter.value = meter
  }
}

// 拖拽離開節點
const handleDragLeave = (event) => {
  event.preventDefault()
  event.stopPropagation()
  // 只有當離開整個元素時才清除，不在子元素間切換時清除
  const rect = event.currentTarget.getBoundingClientRect()
  const x = event.clientX
  const y = event.clientY
  
  if (x < rect.left || x >= rect.right || y < rect.top || y >= rect.bottom) {
    dropTargetMeter.value = null
  }
}

// 放下到節點上（成為其子節點）
const handleDropOnMeter = async (event, targetMeter) => {
  event.preventDefault()
  event.stopPropagation()
  
  console.log('拖拽放下:', { 
    draggedMeter: draggedMeter.value, 
    targetMeter: targetMeter 
  })
  
  if (!draggedMeter.value || draggedMeter.value.id === targetMeter.id) {
    console.log('取消操作: 無拖拽節點或拖到自己身上')
    return
  }
  
  const movedNode = draggedMeter.value
  // 使用目標節點的 ID 作為新的父節點，成為其子節點
  const newParentId = targetMeter.id
  
  // 獲取該節點的所有子孫節點數量（用於顯示）
  const allDescendantIds = getAllDescendantIds(movedNode.id)
  const descendantCount = allDescendantIds.length - 1 // 減去自己
  
  console.log('準備移動:', {
    nodeId: movedNode.id,
    nodeName: movedNode.name,
    currentParentId: movedNode.parent_id,
    targetNodeId: targetMeter.id,
    targetNodeName: targetMeter.name,
    newParentId: newParentId,
    descendantCount: descendantCount,
    willMoveWithDescendants: descendantCount > 0
  })
  
  // 檢查是否已經是該節點的子節點
  if (movedNode.parent_id === newParentId) {
    console.log('節點已經是該節點的子節點')
    alert('節點已經在該父節點下，無需移動')
    draggedMeter.value = null
    dropTargetMeter.value = null
    return
  }
  
  // 檢查是否會造成循環引用（不能將父節點拖到子節點下）
  if (isDescendant(targetMeter.id, movedNode.id)) {
    alert('不能將節點移動到其子孫節點下')
    draggedMeter.value = null
    dropTargetMeter.value = null
    return
  }
  
  try {
    // 只移動根節點本身，保持子孫節點的相對層級關係
    // 子孫節點的 parent_id 保持不變，這樣可以保留原始的樹狀結構
    const requestBody = {
      target_parent_id: newParentId,
      node_ids: [movedNode.id] // 只移動根節點
    }
    
    console.log('發送 API 請求（只移動根節點，子孫保持相對關係）:', requestBody)
    console.log('子孫節點將自動跟隨:', descendantCount > 0 ? allDescendantIds.slice(1) : '無')
    
    const response = await fetch('http://localhost:3001/nodes/move', {
      method: 'PATCH',
      headers: {
        'Content-Type': 'application/json'
      },
      body: JSON.stringify(requestBody)
    })
    
    console.log('API 響應狀態:', response.status)
    
    const result = await response.json()
    console.log('API 響應:', result)
    
    if (!response.ok || !result.success) {
      throw new Error(result.error || '移動節點失敗')
    }
    
    // API 成功後，重新載入資料
    await fetchNodes()
    
    const message = descendantCount > 0 
      ? `已將 "${movedNode.name}" 及其 ${descendantCount} 個子孫節點移動到 "${targetMeter.name}" 的子層級下`
      : `已將 "${movedNode.name}" 移動到 "${targetMeter.name}" 的子層級下`
    alert(message)
  } catch (error) {
    console.error('移動節點失敗:', error)
    alert(`移動節點失敗: ${error.message}`)
  }
  
  draggedMeter.value = null
  dropTargetMeter.value = null
}

// 檢查 targetId 是否是 nodeId 的後代節點
const isDescendant = (targetId, nodeId) => {
  const children = getChildNodes(nodeId)
  for (const child of children) {
    if (child.id === targetId) return true
    if (isDescendant(targetId, child.id)) return true
  }
  return false
}

// 獲取節點及其所有子孫節點的 ID
const getAllDescendantIds = (nodeId) => {
  const ids = [nodeId]
  const children = getChildNodes(nodeId)
  
  for (const child of children) {
    ids.push(...getAllDescendantIds(child.id))
  }
  
  return ids
}

// 放下（舊的層級拖拽，保留以防需要）
const handleDrop = (event, targetLayer) => {
  event.preventDefault()
  
  if (draggedMeter.value && draggedFromLayer.value !== null) {
    const meter = draggedMeter.value
    const fromLayer = draggedFromLayer.value
    
    // 如果拖拽到不同層級
    if (fromLayer !== targetLayer) {
      // 從原層級移除
      const index = layers.value[fromLayer].findIndex(m => m.id === meter.id)
      if (index > -1) {
        layers.value[fromLayer].splice(index, 1)
      }
      
      // 加入新層級
      meter.layer = targetLayer
      meter.selected = false // 移動到新層級時取消選擇
      layers.value[targetLayer].push(meter)
      
      // 檢查原層級是否還有選中的電表
      const hasSelected = layers.value[fromLayer].some(m => m.selected)
      if (!hasSelected && activeLayer.value === fromLayer) {
        activeLayer.value = null
      }
    }
    
    draggedMeter.value = null
    draggedFromLayer.value = null
  }
}

// 編輯電表
const editMeters = () => {
  let selectedMeters = []
  Object.values(layers.value).forEach(layerMeters => {
    selectedMeters = selectedMeters.concat(layerMeters.filter(m => m.selected))
  })
  
  if (selectedMeters.length === 0) {
    alert('請先選擇要編輯的電表')
    return
  }
  alert(`編輯 ${selectedMeters.length} 個電表`)
  // 這裡可以加入編輯邏輯
}

// 移到根目錄
const moveToRoot = async () => {
  let selectedMeters = []
  Object.values(layers.value).forEach(layerMeters => {
    selectedMeters = selectedMeters.concat(layerMeters.filter(m => m.selected))
  })
  
  if (selectedMeters.length === 0) {
    alert('請先選擇要移動的電表')
    return
  }
  
  // 過濾出已經在根目錄的項目
  const alreadyRoot = selectedMeters.filter(m => m.parent_id === null)
  const needMove = selectedMeters.filter(m => m.parent_id !== null)
  
  if (needMove.length === 0) {
    alert('所選項目已在根目錄')
    return
  }
  
  const confirmMsg = alreadyRoot.length > 0
    ? `將移動 ${needMove.length} 個電表到根目錄（${alreadyRoot.length} 個已在根目錄）\n確定要繼續嗎？`
    : `確定要將 ${needMove.length} 個電表移動到根目錄嗎？`
  
  if (!confirm(confirmMsg)) {
    return
  }
  
  try {
    // 收集所有需要移動的節點ID
    const nodeIds = needMove.map(m => m.id)
    
    console.log('移動到根目錄:', {
      nodeIds,
      count: nodeIds.length
    })
    
    const requestBody = {
      target_parent_id: null, // null 表示根目錄
      node_ids: nodeIds
    }
    
    console.log('發送 API 請求（移到根目錄）:', requestBody)
    
    const response = await fetch('http://localhost:3001/nodes/move', {
      method: 'PATCH',
      headers: {
        'Content-Type': 'application/json'
      },
      body: JSON.stringify(requestBody)
    })
    
    console.log('API 響應狀態:', response.status)
    
    const result = await response.json()
    console.log('API 響應:', result)
    
    if (!response.ok || !result.success) {
      throw new Error(result.error || '移動到根目錄失敗')
    }
    
    // API 成功後，重新載入資料
    await fetchNodes()
    
    alert(`已將 ${needMove.length} 個電表移動到根目錄`)
  } catch (error) {
    console.error('移動到根目錄失敗:', error)
    alert(`移動到根目錄失敗: ${error.message}`)
  }
}

// 第二個按鈕功能
const secondAction = () => {
  alert('第二個按鈕功能')
}
</script>

<template>
  <div class="power-system">
    <h1>電表紀錄系統</h1>
    
    <!-- 頂部操作欄 -->
    <div class="action-bar">
      <div class="selected-info">
        已選擇 {{ selectedCount }} 電表
      </div>
      <div class="action-buttons">
        <button @click="editMeters" class="btn-edit">編輯電表</button>
        <button @click="moveToRoot" class="btn-move-root" :disabled="selectedCount === 0">移到根目錄</button>
        <button @click="secondAction" class="btn-secondary">其他操作</button>
      </div>
    </div>

    <!-- 分層電表列表 -->
    <div class="layers-container">
      <!-- 樹狀結構 -->
      <div class="layer-section">
        <div class="layer-header">
          <h2>電表樹狀結構</h2>
        </div>
        
        <div class="meter-list">
          <MeterRow 
            v-for="meter in layers[0]" 
            :key="meter.id"
            :meter="meter" 
            :depth="0"
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
          
          <div v-if="layers[0].length === 0" class="empty-layer">
            載入中或無電表資料...
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.power-system {
  max-width: 1400px;
  margin: 0 auto;
  padding: 20px;
}

h1 {
  color: #2c3e50;
  margin-bottom: 30px;
  text-align: center;
}

/* 頂部操作欄 */
.action-bar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  background: #f8f9fa;
  padding: 15px 20px;
  border-radius: 8px;
  margin-bottom: 30px;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
}

.selected-info {
  font-size: 18px;
  font-weight: 600;
  color: #2c3e50;
}

.action-buttons {
  display: flex;
  gap: 10px;
}

button {
  padding: 10px 20px;
  border: none;
  border-radius: 6px;
  font-size: 14px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.3s ease;
}

.btn-edit {
  background: #3498db;
  color: white;
}

.btn-edit:hover {
  background: #2980b9;
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(52, 152, 219, 0.3);
}

.btn-move-root {
  background: #27ae60;
  color: white;
}

.btn-move-root:hover:not(:disabled) {
  background: #229954;
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(39, 174, 96, 0.3);
}

.btn-move-root:disabled {
  background: #bdc3c7;
  cursor: not-allowed;
  opacity: 0.6;
}

.btn-secondary {
  background: #95a5a6;
  color: white;
}

.btn-secondary:hover {
  background: #7f8c8d;
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(149, 165, 166, 0.3);
}

/* 分層容器 */
.layers-container {
  display: flex;
  flex-direction: column;
  gap: 25px;
}

.layer-section {
  background: #ffffff;
  border-radius: 12px;
  padding: 20px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
  min-height: 150px;
  transition: all 0.3s ease;
}

.layer-section:hover {
  box-shadow: 0 4px 12px rgba(0,0,0,0.15);
}

.layer-child {
  margin-left: 30px;
  border-left: 3px solid #3498db;
  animation: slideDown 0.3s ease-out;
}

@keyframes slideDown {
  from {
    opacity: 0;
    transform: translateY(-10px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.layer-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 15px;
  padding-bottom: 10px;
  border-bottom: 2px solid #e0e0e0;
}

.layer-header h2 {
  margin: 0;
  color: #2c3e50;
  font-size: 20px;
  font-weight: 600;
}

.meter-count {
  color: #7f8c8d;
  font-size: 14px;
  font-weight: 500;
}

/* 電表列表 */
.meter-list {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.empty-layer {
  padding: 40px;
  text-align: center;
  color: #95a5a6;
  font-style: italic;
  border: 2px dashed #e0e0e0;
  border-radius: 8px;
  background: #fafafa;
}
</style>
