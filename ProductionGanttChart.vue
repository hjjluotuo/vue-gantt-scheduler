<template>
  <div class="gantt-container">
    <div class="gantt-header">
      <h2 class="text-xl font-bold mb-4">生产排程甘特图</h2>
      <div class="flex gap-2">
        <button @click="addTask" class="btn-primary">添加任务</button>
        <button @click="saveSchedule" class="btn-secondary">保存排程</button>
      </div>
    </div>
    
    <!-- 时间轴标题 -->
    <div class="gantt-timeline-header" :style="{ width: `${timelineWidth}px` }">
      <div class="timeline-header-spacer" :style="{ width: `${resourceColumnWidth}px` }"></div>
      <div class="timeline-header-cells">
        <div 
          v-for="day in visibleDays" 
          :key="day.date" 
          class="timeline-header-cell"
          :style="{ width: `${dayWidth}px` }"
        >
          {{ formatDate(day.date) }}
        </div>
      </div>
    </div>
    
    <div class="gantt-body" ref="ganttBody">
      <!-- 资源列表 -->
      <div class="gantt-resources">
        <div class="resource-header">资源/设备</div>
        <div 
          v-for="resource in resources" 
          :key="resource.id" 
          class="resource-item"
        >
          {{ resource.name }}
        </div>
      </div>
      
      <!-- 甘特图主体 -->
      <div 
        class="gantt-chart" 
        ref="ganttChart"
        :style="{ width: `${timelineWidth}px` }"
        @mousedown="handleChartMouseDown"
      >
        <!-- 网格背景 -->
        <div class="gantt-grid">
          <div 
            v-for="resource in resources" 
            :key="`grid-${resource.id}`" 
            class="gantt-grid-row"
          >
            <div 
              v-for="day in visibleDays" 
              :key="`grid-${resource.id}-${day.date}`" 
              class="gantt-grid-cell"
              :class="{ 'weekend': isWeekend(day.date) }"
              :style="{ width: `${dayWidth}px` }"
            ></div>
          </div>
        </div>
        
        <!-- 任务条 -->
        <div 
          v-for="task in tasks" 
          :key="task.id" 
          class="gantt-task"
          :class="{ 
            'selected': selectedTask?.id === task.id,
            'conflicting': conflictingTasks.includes(task.id)
          }"
          :style="{
            left: `${getTaskLeft(task)}px`,
            top: `${getTaskTop(task)}px`,
            width: `${getTaskWidth(task)}px`,
            height: `${rowHeight - 4}px`,
            backgroundColor: task.color || '#4299e1'
          }"
          @mousedown.stop="handleTaskMouseDown($event, task)"
        >
          <div class="task-content">{{ task.name }}</div>
          <!-- 左侧调整手柄 -->
          <div 
            class="task-resize-handle left"
            @mousedown.stop="handleResizeStart($event, task, 'left')"
          ></div>
          <!-- 右侧调整手柄 -->
          <div 
            class="task-resize-handle right"
            @mousedown.stop="handleResizeStart($event, task, 'right')"
          ></div>
        </div>
      </div>
    </div>
    
    <!-- 任务详情面板 -->
    <div v-if="selectedTask" class="task-details-panel">
      <h3 class="text-lg font-semibold mb-2">任务详情</h3>
      <div class="mb-2">
        <label class="block text-sm mb-1">名称:</label>
        <input v-model="selectedTask.name" class="input-field" />
      </div>
      <div class="mb-2">
        <label class="block text-sm mb-1">开始日期:</label>
        <input 
          type="date" 
          :value="formatDateForInput(selectedTask.startDate)" 
          @input="updateTaskDate($event, 'startDate')"
          class="input-field" 
        />
      </div>
      <div class="mb-2">
        <label class="block text-sm mb-1">结束日期:</label>
        <input 
          type="date" 
          :value="formatDateForInput(selectedTask.endDate)" 
          @input="updateTaskDate($event, 'endDate')"
          class="input-field" 
        />
      </div>
      <div class="mb-2">
        <label class="block text-sm mb-1">资源:</label>
        <select 
          v-model="selectedTask.resourceId" 
          class="input-field"
          @change="handleResourceChange"
        >
          <option v-for="resource in resources" :key="resource.id" :value="resource.id">
            {{ resource.name }}
          </option>
        </select>
      </div>
      <div class="flex gap-2 mt-4">
        <button @click="closeTaskDetails" class="btn-secondary">关闭</button>
        <button @click="deleteTask" class="btn-danger">删除任务</button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'

// 配置参数
const dayWidth = 60 // 每天的宽度（像素）
const rowHeight = 40 // 每行的高度（像素）
const resourceColumnWidth = 120 // 资源列宽度
const startDate = new Date('2024-01-01') // 甘特图开始日期
const endDate = new Date('2024-01-31') // 甘特图结束日期

// 状态变量
const selectedTask = ref(null)
const isDragging = ref(false)
const isResizing = ref(false)
const resizeDirection = ref(null)
const dragStartX = ref(0)
const dragStartY = ref(0)
const originalLeft = ref(0)
const originalTop = ref(0)
const originalWidth = ref(0)
const ganttChart = ref(null)
const ganttBody = ref(null)
const conflictingTasks = ref([]) // 存储冲突任务的ID

// 资源数据
const resources = ref([
  { id: 1, name: '生产线 A' },
  { id: 2, name: '生产线 B' },
  { id: 3, name: '生产线 C' },
  { id: 4, name: '生产线 D' },
  { id: 5, name: '生产线 E' }
])

// 任务数据
const tasks = ref([
  { 
    id: 1, 
    name: '产品 A 生产', 
    startDate: new Date('2024-01-03'), 
    endDate: new Date('2024-01-07'), 
    resourceId: 1,
    color: '#4299e1'
  },
  { 
    id: 2, 
    name: '产品 B 生产', 
    startDate: new Date('2024-01-05'), 
    endDate: new Date('2024-01-10'), 
    resourceId: 2,
    color: '#48bb78'
  },
  { 
    id: 3, 
    name: '产品 C 生产', 
    startDate: new Date('2024-01-08'), 
    endDate: new Date('2024-01-15'), 
    resourceId: 3,
    color: '#ed8936'
  }
])

// 计算可见天数
const visibleDays = computed(() => {
  const days = []
  const currentDate = new Date(startDate)
  
  while (currentDate <= endDate) {
    days.push({
      date: new Date(currentDate),
      dayOfWeek: currentDate.getDay()
    })
    currentDate.setDate(currentDate.getDate() + 1)
  }
  
  return days
})

// 计算时间轴总宽度
const timelineWidth = computed(() => {
  return visibleDays.value.length * dayWidth + resourceColumnWidth
})

// 格式化日期显示
const formatDate = (date) => {
  return `${date.getMonth() + 1}/${date.getDate()}`
}

// 格式化日期为输入框格式
const formatDateForInput = (date) => {
  const d = new Date(date)
  const year = d.getFullYear()
  const month = String(d.getMonth() + 1).padStart(2, '0')
  const day = String(d.getDate()).padStart(2, '0')
  return `${year}-${month}-${day}`
}

// 判断是否为周末
const isWeekend = (date) => {
  const day = date.getDay()
  return day === 0 || day === 6
}

// 计算任务左侧位置
const getTaskLeft = (task) => {
  const diffDays = differenceInDays(task.startDate, startDate)
  return diffDays * dayWidth + resourceColumnWidth
}

// 计算任务顶部位置
const getTaskTop = (task) => {
  const resourceIndex = resources.value.findIndex(r => r.id === task.resourceId)
  return resourceIndex * rowHeight
}

// 计算任务宽度
const getTaskWidth = (task) => {
  const durationDays = differenceInDays(task.endDate, task.startDate) + 1
  return durationDays * dayWidth
}

// 计算两个日期之间的天数差
const differenceInDays = (date1, date2) => {
  const d1 = new Date(date1)
  const d2 = new Date(date2)
  d1.setHours(0, 0, 0, 0)
  d2.setHours(0, 0, 0, 0)
  return Math.round((d1 - d2) / (24 * 60 * 60 * 1000))
}

// 检查两个任务是否重叠
const tasksOverlap = (task1, task2) => {
  if (task1.id === task2.id) return false
  if (task1.resourceId !== task2.resourceId) return false
  
  const task1Start = new Date(task1.startDate).getTime()
  const task1End = new Date(task1.endDate).getTime()
  const task2Start = new Date(task2.startDate).getTime()
  const task2End = new Date(task2.endDate).getTime()
  
  return (task1Start <= task2End && task1End >= task2Start)
}

// 获取与指定任务重叠的任务列表
const getOverlappingTasks = (task) => {
  return tasks.value.filter(t => tasksOverlap(task, t))
}

// 处理任务重叠冲突
const resolveTaskConflicts = (task) => {
  // 清除之前的冲突标记
  conflictingTasks.value = []
  
  // 获取与当前任务相同资源的所有任务，并按开始日期排序
  const resourceTasks = tasks.value
    .filter(t => t.resourceId === task.resourceId && t.id !== task.id)
    .sort((a, b) => new Date(a.startDate) - new Date(b.startDate))
  
  if (resourceTasks.length === 0) return
  
  // 检查并解决冲突
  let hasConflict = true
  let maxIterations = 100 // 防止无限循环
  let iterations = 0
  
  while (hasConflict && iterations < maxIterations) {
    hasConflict = false
    iterations++
    
    // 按开始日期排序所有任务
    const sortedTasks = [...resourceTasks].sort((a, b) => 
      new Date(a.startDate) - new Date(b.startDate)
    )
    
    for (let i = 0; i < sortedTasks.length; i++) {
      const currentTask = sortedTasks[i]
      
      // 检查当前任务与拖动的任务是否重叠
      if (tasksOverlap(currentTask, task)) {
        hasConflict = true
        conflictingTasks.value.push(currentTask.id)
        
        // 计算需要移动的天数
        const taskEndDate = new Date(task.endDate)
        const daysDiff = differenceInDays(taskEndDate, currentTask.startDate) + 1
        
        // 移动冲突的任务
        const newStartDate = new Date(taskEndDate)
        newStartDate.setDate(newStartDate.getDate() + 1)
        
        const duration = differenceInDays(currentTask.endDate, currentTask.startDate)
        const newEndDate = new Date(newStartDate)
        newEndDate.setDate(newStartDate.getDate() + duration)
        
        // 更新任务日期
        currentTask.startDate = newStartDate
        currentTask.endDate = newEndDate
        
        // 递归检查这个移动是否导致了新的冲突
        resolveTaskConflicts(currentTask)
      }
    }
  }
}

// 处理任务鼠标按下事件
const handleTaskMouseDown = (event, task) => {
  event.preventDefault()
  selectedTask.value = task
  isDragging.value = true
  isResizing.value = false
  
  dragStartX.value = event.clientX
  dragStartY.value = event.clientY
  originalLeft.value = getTaskLeft(task)
  originalTop.value = getTaskTop(task)
  
  // 存储任务的原始日期，用于撤销
  task._originalStartDate = new Date(task.startDate)
  task._originalEndDate = new Date(task.endDate)
  task._originalResourceId = task.resourceId
  
  document.addEventListener('mousemove', handleMouseMove)
  document.addEventListener('mouseup', handleMouseUp)
}

// 处理调整大小开始事件
const handleResizeStart = (event, task, direction) => {
  event.preventDefault()
  selectedTask.value = task
  isResizing.value = true
  isDragging.value = false
  resizeDirection.value = direction
  
  dragStartX.value = event.clientX
  originalLeft.value = getTaskLeft(task)
  originalWidth.value = getTaskWidth(task)
  
  // 存储任务的原始日期，用于撤销
  task._originalStartDate = new Date(task.startDate)
  task._originalEndDate = new Date(task.endDate)
  
  document.addEventListener('mousemove', handleMouseMove)
  document.addEventListener('mouseup', handleMouseUp)
}

// 处理鼠标移动事件
const handleMouseMove = (event) => {
  if (!isDragging.value && !isResizing.value) return
  
  const deltaX = event.clientX - dragStartX.value
  
  if (isResizing.value) {
    // 调整任务大小
    if (resizeDirection.value === 'right') {
      // 调整右侧（改变宽度）
      const newWidth = Math.max(dayWidth, originalWidth.value + deltaX)
      const durationDays = Math.round(newWidth / dayWidth)
      
      const newEndDate = new Date(selectedTask.value.startDate)
      newEndDate.setDate(newEndDate.getDate() + durationDays - 1)
      
      // 确保不超出甘特图时间范围
      if (newEndDate <= endDate) {
        selectedTask.value.endDate = newEndDate
        // 检查并解决冲突
        resolveTaskConflicts(selectedTask.value)
      }
    } else {
      // 调整左侧（改变开始日期和宽度）
      const maxLeftDelta = originalWidth.value - dayWidth
      const constrainedDeltaX = Math.max(-originalLeft.value + resourceColumnWidth, Math.min(maxLeftDelta, deltaX))
      
      const daysDelta = Math.round(constrainedDeltaX / dayWidth)
      
      const newStartDate = new Date(selectedTask.value._originalStartDate)
      newStartDate.setDate(newStartDate.getDate() - daysDelta)
      
      // 确保不超出甘特图时间范围
      if (newStartDate >= startDate) {
        selectedTask.value.startDate = newStartDate
        
        // 保持任务持续时间不变
        const duration = differenceInDays(selectedTask.value._originalEndDate, selectedTask.value._originalStartDate)
        const newEndDate = new Date(newStartDate)
        newEndDate.setDate(newStartDate.getDate() + duration)
        selectedTask.value.endDate = newEndDate
        
        // 检查并解决冲突
        resolveTaskConflicts(selectedTask.value)
      }
    }
  } else if (isDragging.value) {
    // 拖动任务
    const deltaY = event.clientY - dragStartY.value
    
    // 水平移动（改变日期）
    const daysDelta = Math.round(deltaX / dayWidth)
    const newStartDate = new Date(selectedTask.value._originalStartDate)
    newStartDate.setDate(newStartDate.getDate() + daysDelta)
    
    const duration = differenceInDays(selectedTask.value._originalEndDate, selectedTask.value._originalStartDate)
    const newEndDate = new Date(newStartDate)
    newEndDate.setDate(newStartDate.getDate() + duration)
    
    // 确保不超出甘特图时间范围
    if (newStartDate >= startDate && newEndDate <= endDate) {
      selectedTask.value.startDate = newStartDate
      selectedTask.value.endDate = newEndDate
    }
    
    // 垂直移动（改变资源）
    const rowDelta = Math.round(deltaY / rowHeight)
    const currentResourceIndex = resources.value.findIndex(r => r.id === selectedTask.value._originalResourceId)
    const newResourceIndex = Math.max(0, Math.min(resources.value.length - 1, currentResourceIndex + rowDelta))
    selectedTask.value.resourceId = resources.value[newResourceIndex].id
    
    // 检查并解决冲突
    resolveTaskConflicts(selectedTask.value)
  }
}

// 处理鼠标松开事件
const handleMouseUp = () => {
  isDragging.value = false
  isResizing.value = false
  
  // 清除临时存储的原始日期
  if (selectedTask.value) {
    delete selectedTask.value._originalStartDate
    delete selectedTask.value._originalEndDate
    delete selectedTask.value._originalResourceId
  }
  
  document.removeEventListener('mousemove', handleMouseMove)
  document.removeEventListener('mouseup', handleMouseUp)
  
  // 清除冲突标记
  setTimeout(() => {
    conflictingTasks.value = []
  }, 1000)
}

// 处理甘特图区域点击事件
const handleChartMouseDown = (event) => {
  // 如果点击的是甘特图背景，取消选择任务
  if (event.target === ganttChart.value || event.target.classList.contains('gantt-grid-cell')) {
    selectedTask.value = null
  }
}

// 更新任务日期
const updateTaskDate = (event, dateType) => {
  if (selectedTask.value) {
    const oldDate = new Date(selectedTask.value[dateType])
    selectedTask.value[dateType] = new Date(event.target.value)
    
    // 如果是开始日期变更，保持任务持续时间不变
    if (dateType === 'startDate') {
      const duration = differenceInDays(selectedTask.value.endDate, oldDate)
      const newEndDate = new Date(selectedTask.value.startDate)
      newEndDate.setDate(newEndDate.getDate() + duration)
      selectedTask.value.endDate = newEndDate
    }
    
    // 检查并解决冲突
    resolveTaskConflicts(selectedTask.value)
  }
}

// 处理资源变更
const handleResourceChange = () => {
  if (selectedTask.value) {
    // 检查并解决冲突
    resolveTaskConflicts(selectedTask.value)
  }
}

// 添加新任务
const addTask = () => {
  const newTaskId = Math.max(0, ...tasks.value.map(t => t.id)) + 1
  const newTask = {
    id: newTaskId,
    name: '新任务',
    startDate: new Date(startDate),
    endDate: new Date(startDate),
    resourceId: resources.value[0].id,
    color: getRandomColor()
  }
  
  // 设置默认结束日期为开始日期后3天
  newTask.endDate.setDate(newTask.startDate.getDate() + 3)
  
  tasks.value.push(newTask)
  selectedTask.value = newTask
  
  // 检查并解决冲突
  resolveTaskConflicts(newTask)
}

// 删除任务
const deleteTask = () => {
  if (selectedTask.value) {
    const index = tasks.value.findIndex(t => t.id === selectedTask.value.id)
    if (index !== -1) {
      tasks.value.splice(index, 1)
      selectedTask.value = null
    }
  }
}

// 关闭任务详情面板
const closeTaskDetails = () => {
  selectedTask.value = null
}

// 保存排程
const saveSchedule = () => {
  console.log('保存排程:', tasks.value)
  alert('排程已保存！')
}

// 生成随机颜色
const getRandomColor = () => {
  const colors = ['#4299e1', '#48bb78', '#ed8936', '#9f7aea', '#f56565', '#38b2ac']
  return colors[Math.floor(Math.random() * colors.length)]
}

// 生命周期钩子
onMounted(() => {
  document.addEventListener('keydown', handleKeyDown)
})

onUnmounted(() => {
  document.removeEventListener('keydown', handleKeyDown)
  document.removeEventListener('mousemove', handleMouseMove)
  document.removeEventListener('mouseup', handleMouseUp)
})

// 处理键盘事件
const handleKeyDown = (event) => {
  if (event.key === 'Escape') {
    selectedTask.value = null
  } else if (event.key === 'Delete' && selectedTask.value) {
    deleteTask()
  }
}
</script>

<style scoped>
.gantt-container {
  display: flex;
  flex-direction: column;
  height: 100%;
  font-family: 'Inter', sans-serif;
  color: #1a202c;
  background-color: #f7fafc;
  border-radius: 8px;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
  overflow: hidden;
}

.gantt-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 16px;
  background-color: white;
  border-bottom: 1px solid #e2e8f0;
}

.gantt-timeline-header {
  display: flex;
  position: sticky;
  top: 0;
  z-index: 10;
  background-color: white;
  border-bottom: 1px solid #e2e8f0;
}

.timeline-header-spacer {
  flex-shrink: 0;
  border-right: 1px solid #e2e8f0;
}

.timeline-header-cells {
  display: flex;
}

.timeline-header-cell {
  flex-shrink: 0;
  padding: 8px 4px;
  text-align: center;
  font-size: 0.875rem;
  font-weight: 500;
  border-right: 1px solid #e2e8f0;
}

.gantt-body {
  display: flex;
  overflow: auto;
  flex: 1;
}

.gantt-resources {
  position: sticky;
  left: 0;
  z-index: 5;
  background-color: white;
  width: 120px;
  flex-shrink: 0;
  border-right: 1px solid #e2e8f0;
}

.resource-header {
  height: 36px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: 600;
  font-size: 0.875rem;
  border-bottom: 1px solid #e2e8f0;
  background-color: #f8fafc;
}

.resource-item {
  height: 40px;
  display: flex;
  align-items: center;
  padding: 0 8px;
  border-bottom: 1px solid #e2e8f0;
  font-size: 0.875rem;
}

.gantt-chart {
  position: relative;
  flex: 1;
}

.gantt-grid {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  z-index: 1;
}

.gantt-grid-row {
  display: flex;
  height: 40px;
  border-bottom: 1px solid #e2e8f0;
}

.gantt-grid-cell {
  flex-shrink: 0;
  height: 100%;
  border-right: 1px solid #e2e8f0;
}

.gantt-grid-cell.weekend {
  background-color: #f1f5f9;
}

.gantt-task {
  position: absolute;
  z-index: 2;
  border-radius: 4px;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.1);
  cursor: move;
  overflow: hidden;
  transition: box-shadow 0.2s, transform 0.2s;
}

.gantt-task:hover {
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
}

.gantt-task.selected {
  box-shadow: 0 0 0 2px #3182ce;
  z-index: 3;
}

.gantt-task.conflicting {
  animation: pulse 1s infinite;
}

@keyframes pulse {
  0% {
    transform: scale(1);
  }
  50% {
    transform: scale(1.02);
  }
  100% {
    transform: scale(1);
  }
}

.task-content {
  padding: 4px 8px;
  font-size: 0.8125rem;
  color: white;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.task-resize-handle {
  position: absolute;
  top: 0;
  width: 6px;
  height: 100%;
  cursor: ew-resize;
}

.task-resize-handle.left {
  left: 0;
}

.task-resize-handle.right {
  right: 0;
}

.task-details-panel {
  position: absolute;
  top: 16px;
  right: 16px;
  width: 300px;
  background-color: white;
  border-radius: 8px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  padding: 16px;
  z-index: 100;
}

.input-field {
  width: 100%;
  padding: 6px 8px;
  border: 1px solid #e2e8f0;
  border-radius: 4px;
  font-size: 0.875rem;
}

.btn-primary {
  background-color: #3182ce;
  color: white;
  padding: 6px 12px;
  border-radius: 4px;
  font-size: 0.875rem;
  font-weight: 500;
  cursor: pointer;
  transition: background-color 0.2s;
}

.btn-primary:hover {
  background-color: #2c5282;
}

.btn-secondary {
  background-color: #e2e8f0;
  color: #4a5568;
  padding: 6px 12px;
  border-radius: 4px;
  font-size: 0.875rem;
  font-weight: 500;
  cursor: pointer;
  transition: background-color 0.2s;
}

.btn-secondary:hover {
  background-color: #cbd5e0;
}

.btn-danger {
  background-color: #f56565;
  color: white;
  padding: 6px 12px;
  border-radius: 4px;
  font-size: 0.875rem;
  font-weight: 500;
  cursor: pointer;
  transition: background-color 0.2s;
}

.btn-danger:hover {
  background-color: #e53e3e;
}
</style>
