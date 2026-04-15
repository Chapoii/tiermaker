<template>
  <div class="app">
    <!-- 导航 -->
    <nav class="navbar">
      <div class="nav-inner">
        <div class="nav-logo">🏆 TierMaker</div>
        <div class="nav-actions">
          <button class="btn btn-outline" @click="resetAll">🔄 重置</button>
          <button class="btn btn-primary" @click="exportImage">📸 导出图片</button>
        </div>
      </div>
    </nav>

    <!-- 标题区 -->
    <header class="header">
      <h1 class="title">等级排序工具</h1>
      <p class="subtitle">自定义等级梯度，拖拽排序你的项目</p>
    </header>

    <!-- 主内容 -->
    <main class="main">
      <!-- 左侧：待排序项目池 -->
      <aside class="item-pool">
        <div class="pool-header">
          <h2>📦 待排序项目</h2>
          <div class="pool-actions">
            <input
              v-model="newItemName"
              class="input"
              placeholder="输入项目名称..."
              @keyup.enter="addItem"
            />
            <button class="btn btn-sm btn-primary" @click="addItem">+ 添加</button>
          </div>
          <div class="pool-batch">
            <textarea
              v-model="batchItems"
              class="textarea"
              placeholder="批量添加（每行一个）"
              rows="3"
            ></textarea>
            <button class="btn btn-sm btn-outline" @click="batchAdd">批量添加</button>
          </div>
        </div>
        <div class="pool-items">
          <div
            v-for="(item, index) in unrankedItems"
            :key="item.id"
            class="pool-item"
            draggable="true"
            @dragstart="onPoolDragStart($event, index)"
          >
            <span class="item-name">{{ item.name }}</span>
            <button class="btn-icon" @click="removeItem(item.id)">×</button>
          </div>
          <div v-if="unrankedItems.length === 0" class="empty-hint">
            没有待排序的项目了
          </div>
        </div>
      </aside>

      <!-- 右侧：等级排序区 -->
      <section class="tier-board" ref="tierBoardRef">
        <!-- 等级行 -->
        <div
          v-for="(tier, tIndex) in tiers"
          :key="tier.id"
          class="tier-row"
          :style="{ '--tier-color': tier.color }"
        >
          <div class="tier-label" :style="{ background: tier.color }">
            <span class="tier-name" contenteditable="true" @blur="updateTierName(tIndex, $event)">{{ tier.name }}</span>
            <div class="tier-label-actions">
              <input type="color" :value="tier.color" @input="updateTierColor(tIndex, $event)" class="color-picker" />
              <button class="btn-icon-sm" @click="removeTier(tIndex)" title="删除此等级">×</button>
            </div>
          </div>
          <div
            class="tier-items"
            @dragover.prevent="onTierDragOver($event, tIndex)"
            @drop="onTierDrop($event, tIndex)"
            @dragleave="onTierDragLeave($event)"
          >
            <div
              v-for="(item, iIndex) in getTierItems(tier.id)"
              :key="item.id"
              class="tier-item"
              draggable="true"
              @dragstart="onTierItemDragStart($event, tIndex, iIndex)"
              @dragover.prevent
              @drop.stop="onTierItemDrop($event, tIndex, iIndex)"
            >
              <span class="item-name">{{ item.name }}</span>
              <button class="btn-icon" @click="unrankItem(item.id)">×</button>
            </div>
            <div v-if="getTierItems(tier.id).length === 0" class="tier-empty">
              拖拽项目到此处
            </div>
          </div>
        </div>

        <!-- 添加等级 -->
        <div class="add-tier-row">
          <div class="add-tier-controls">
            <input v-model="newTierName" class="input input-sm" placeholder="等级名称..." />
            <input type="color" v-model="newTierColor" class="color-picker" />
            <button class="btn btn-sm btn-outline" @click="addTier">+ 添加等级</button>
          </div>
        </div>
      </section>
    </main>

    <!-- 预设模板 -->
    <section class="presets">
      <h3>📋 快速预设</h3>
      <div class="preset-buttons">
        <button class="btn btn-preset" @click="applyPreset('sabc')">S/A/B/C/D</button>
        <button class="btn btn-preset" @click="applyPreset('chinese')">神/强/中/弱/拉</button>
        <button class="btn btn-preset" @click="applyPreset('emoji')">🔥/⭐/👍/😐/👎</button>
        <button class="btn btn-preset" @click="applyPreset('food')">米其林评分</button>
      </div>
    </section>

    <!-- Footer -->
    <footer class="footer">
      <p>© 2026 TierMaker · MIT License · 仅供娱乐</p>
    </footer>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue'

// ==================== 数据 ====================
const tierBoardRef = ref(null)

const tiers = ref([
  { id: 's', name: 'S', color: '#ff7f7f' },
  { id: 'a', name: 'A', color: '#ffbf7f' },
  { id: 'b', name: 'B', color: '#ffff7f' },
  { id: 'c', name: 'C', color: '#bfff7f' },
  { id: 'd', name: 'D', color: '#7fbfff' },
])

const items = ref([])

const newItemName = ref('')
const batchItems = ref('')
const newTierName = ref('')
const newTierColor = ref('#a78bfa')

// 拖拽状态
let dragSource = null // { type: 'pool'|'tier', index, tierIndex? }

// ==================== 计算属性 ====================
const unrankedItems = computed(() => items.value.filter(i => !i.tierId))

function getTierItems(tierId) {
  return items.value.filter(i => i.tierId === tierId)
}

// ==================== 项目操作 ====================
let nextItemId = 1

function addItem() {
  const name = newItemName.value.trim()
  if (!name) return
  items.value.push({ id: nextItemId++, name, tierId: null })
  newItemName.value = ''
}

function batchAdd() {
  const lines = batchItems.value.split('\n').map(l => l.trim()).filter(l => l)
  lines.forEach(name => {
    items.value.push({ id: nextItemId++, name, tierId: null })
  })
  batchItems.value = ''
}

function removeItem(id) {
  items.value = items.value.filter(i => i.id !== id)
}

function unrankItem(id) {
  const item = items.value.find(i => i.id === id)
  if (item) item.tierId = null
}

// ==================== 等级操作 ====================
let nextTierId = 1

function addTier() {
  const name = newTierName.value.trim()
  if (!name) return
  tiers.value.push({
    id: 'tier_' + nextTierId++,
    name,
    color: newTierColor.value,
  })
  newTierName.value = ''
}

function removeTier(index) {
  const tier = tiers.value[index]
  // 将该等级的项目移回待排序
  items.value.forEach(item => {
    if (item.tierId === tier.id) item.tierId = null
  })
  tiers.value.splice(index, 1)
}

function updateTierName(index, event) {
  const name = event.target.textContent.trim()
  if (name) tiers.value[index].name = name
  else event.target.textContent = tiers.value[index].name
}

function updateTierColor(index, event) {
  tiers.value[index].color = event.target.value
}

// ==================== 拖拽：从项目池拖出 ====================
function onPoolDragStart(event, index) {
  dragSource = { type: 'pool', index }
  event.dataTransfer.effectAllowed = 'move'
  event.dataTransfer.setData('text/plain', '')
}

// ==================== 拖拽：放到等级行 ====================
function onTierDragOver(event, tierIndex) {
  event.dataTransfer.dropEffect = 'move'
  event.currentTarget.classList.add('drag-over')
}

function onTierDragLeave(event) {
  event.currentTarget.classList.remove('drag-over')
}

function onTierDrop(event, tierIndex) {
  event.currentTarget.classList.remove('drag-over')
  if (!dragSource) return

  const tierId = tiers.value[tierIndex].id

  if (dragSource.type === 'pool') {
    const item = unrankedItems.value[dragSource.index]
    if (item) item.tierId = tierId
  } else if (dragSource.type === 'tier') {
    // 从另一个等级拖到这个等级
    const srcTierId = tiers.value[dragSource.tierIndex].id
    const srcItems = items.value.filter(i => i.tierId === srcTierId)
    const item = srcItems[dragSource.index]
    if (item) item.tierId = tierId
  }

  dragSource = null
}

// ==================== 拖拽：等级内排序 ====================
function onTierItemDragStart(event, tierIndex, itemIndex) {
  dragSource = { type: 'tier', tierIndex, index: itemIndex }
  event.dataTransfer.effectAllowed = 'move'
  event.dataTransfer.setData('text/plain', '')
}

function onTierItemDrop(event, targetTierIndex, targetIndex) {
  event.stopPropagation()
  if (!dragSource || dragSource.type !== 'tier') return

  const srcTierId = tiers.value[dragSource.tierIndex].id
  const targetTierId = tiers.value[targetTierIndex].id

  // 获取源和目标的 item
  const srcTierItems = items.value.filter(i => i.tierId === srcTierId)
  const movedItem = srcTierItems[dragSource.index]
  if (!movedItem) return

  // 获取目标等级的 items
  const targetTierItems = items.value.filter(i => i.tierId === targetTierId)

  if (srcTierId === targetTierId) {
    // 同一等级内重排
    const [removed] = items.value.splice(items.value.indexOf(movedItem), 1)
    const targetItem = targetTierItems[targetIndex]
    const insertAt = items.value.indexOf(targetItem)
    items.value.splice(insertAt, 0, removed)
  } else {
    // 跨等级移动
    movedItem.tierId = targetTierId
  }

  dragSource = null
}

// ==================== 预设模板 ====================
const presets = {
  sabc: {
    tiers: [
      { name: 'S', color: '#ff7f7f' },
      { name: 'A', color: '#ffbf7f' },
      { name: 'B', color: '#ffff7f' },
      { name: 'C', color: '#bfff7f' },
      { name: 'D', color: '#7fbfff' },
    ],
    items: ['项目A', '项目B', '项目C', '项目D', '项目E', '项目F', '项目G', '项目H'],
  },
  chinese: {
    tiers: [
      { name: '神', color: '#ff6b6b' },
      { name: '强', color: '#ffa94d' },
      { name: '中', color: '#ffd43b' },
      { name: '弱', color: '#69db7c' },
      { name: '拉', color: '#748ffc' },
    ],
    items: ['选项1', '选项2', '选项3', '选项4', '选项5', '选项6'],
  },
  emoji: {
    tiers: [
      { name: '🔥', color: '#ff4500' },
      { name: '⭐', color: '#ffd700' },
      { name: '👍', color: '#32cd32' },
      { name: '😐', color: '#808080' },
      { name: '👎', color: '#4169e1' },
    ],
    items: ['项目A', '项目B', '项目C', '项目D', '项目E'],
  },
  food: {
    tiers: [
      { name: '三星', color: '#ff6b6b' },
      { name: '二星', color: '#ffa94d' },
      { name: '一星', color: '#ffd43b' },
      { name: '必比登', color: '#69db7c' },
      { name: '未上榜', color: '#adb5bd' },
    ],
    items: ['餐厅A', '餐厅B', '餐厅C', '餐厅D', '餐厅E', '餐厅F'],
  },
}

function applyPreset(key) {
  const preset = presets[key]
  if (!preset) return

  tiers.value = preset.tiers.map((t, i) => ({
    id: 'preset_' + i,
    name: t.name,
    color: t.color,
  }))
  nextTierId = 100

  items.value = preset.items.map((name, i) => ({
    id: nextItemId++,
    name,
    tierId: null,
  }))
}

// ==================== 重置 ====================
function resetAll() {
  tiers.value = [
    { id: 's', name: 'S', color: '#ff7f7f' },
    { id: 'a', name: 'A', color: '#ffbf7f' },
    { id: 'b', name: 'B', color: '#ffff7f' },
    { id: 'c', name: 'C', color: '#bfff7f' },
    { id: 'd', name: 'D', color: '#7fbfff' },
  ]
  items.value = []
  nextTierId = 1
  nextItemId = 1
}

// ==================== 导出图片 ====================
function exportImage() {
  alert('提示：请使用浏览器截图功能保存排序结果（推荐使用系统截图工具）')
}
</script>

<style>
@import url('https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600;700;800&display=swap');

* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

:root {
  --bg: #0f0f1a;
  --bg-card: #1a1a2e;
  --bg-hover: #252540;
  --border: #2a2a4a;
  --text: #e8e8f0;
  --text-dim: #8888aa;
  --primary: #6c5ce7;
  --primary-hover: #7c6ef7;
  --radius: 12px;
  --radius-sm: 8px;
}

body {
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, sans-serif;
  background: var(--bg);
  color: var(--text);
  min-height: 100vh;
}

.app {
  max-width: 1400px;
  margin: 0 auto;
  padding: 0 20px;
}

/* 导航 */
.navbar {
  position: sticky;
  top: 0;
  z-index: 100;
  background: rgba(15, 15, 26, 0.9);
  backdrop-filter: blur(12px);
  border-bottom: 1px solid var(--border);
  padding: 12px 0;
}
.nav-inner {
  display: flex;
  justify-content: space-between;
  align-items: center;
}
.nav-logo {
  font-size: 1.3rem;
  font-weight: 800;
  background: linear-gradient(135deg, #6c5ce7, #a78bfa);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
}
.nav-actions {
  display: flex;
  gap: 8px;
}

/* 按钮 */
.btn {
  padding: 8px 16px;
  border-radius: var(--radius-sm);
  border: none;
  font-size: 0.85rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s;
  font-family: inherit;
}
.btn-primary {
  background: var(--primary);
  color: #fff;
}
.btn-primary:hover { background: var(--primary-hover); }
.btn-outline {
  background: transparent;
  color: var(--text-dim);
  border: 1px solid var(--border);
}
.btn-outline:hover { border-color: var(--primary); color: var(--primary); }
.btn-sm { padding: 6px 12px; font-size: 0.8rem; }
.btn-icon {
  background: none;
  border: none;
  color: var(--text-dim);
  cursor: pointer;
  font-size: 1.1rem;
  padding: 2px 6px;
  border-radius: 4px;
  transition: all 0.2s;
}
.btn-icon:hover { color: #ff6b6b; background: rgba(255,107,107,0.1); }
.btn-icon-sm {
  background: none;
  border: none;
  color: rgba(255,255,255,0.6);
  cursor: pointer;
  font-size: 0.9rem;
  padding: 0 4px;
}
.btn-icon-sm:hover { color: #fff; }

/* 输入框 */
.input {
  padding: 8px 12px;
  border-radius: var(--radius-sm);
  border: 1px solid var(--border);
  background: var(--bg);
  color: var(--text);
  font-size: 0.85rem;
  font-family: inherit;
  outline: none;
  transition: border-color 0.2s;
  width: 100%;
}
.input:focus { border-color: var(--primary); }
.input-sm { padding: 6px 10px; font-size: 0.8rem; }
.textarea {
  padding: 8px 12px;
  border-radius: var(--radius-sm);
  border: 1px solid var(--border);
  background: var(--bg);
  color: var(--text);
  font-size: 0.82rem;
  font-family: inherit;
  outline: none;
  resize: vertical;
  width: 100%;
  transition: border-color 0.2s;
}
.textarea:focus { border-color: var(--primary); }

/* 标题 */
.header {
  text-align: center;
  padding: 32px 0 24px;
}
.title {
  font-size: 2rem;
  font-weight: 800;
  background: linear-gradient(135deg, #6c5ce7, #a78bfa, #f472b6);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
}
.subtitle {
  color: var(--text-dim);
  font-size: 0.95rem;
  margin-top: 8px;
}

/* 主布局 */
.main {
  display: flex;
  gap: 24px;
  align-items: flex-start;
}

/* 项目池 */
.item-pool {
  width: 260px;
  flex-shrink: 0;
  background: var(--bg-card);
  border-radius: var(--radius);
  border: 1px solid var(--border);
  overflow: hidden;
  position: sticky;
  top: 72px;
}
.pool-header {
  padding: 16px;
  border-bottom: 1px solid var(--border);
}
.pool-header h2 {
  font-size: 0.95rem;
  font-weight: 700;
  margin-bottom: 12px;
}
.pool-actions {
  display: flex;
  gap: 6px;
  margin-bottom: 10px;
}
.pool-batch {
  display: flex;
  flex-direction: column;
  gap: 6px;
}
.pool-items {
  padding: 8px;
  max-height: 60vh;
  overflow-y: auto;
}
.pool-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 10px 12px;
  background: var(--bg);
  border-radius: var(--radius-sm);
  margin-bottom: 6px;
  cursor: grab;
  transition: all 0.2s;
  border: 1px solid transparent;
}
.pool-item:hover {
  border-color: var(--primary);
  background: var(--bg-hover);
}
.pool-item:active { cursor: grabbing; }
.item-name {
  font-size: 0.85rem;
  font-weight: 500;
}
.empty-hint {
  text-align: center;
  color: var(--text-dim);
  font-size: 0.82rem;
  padding: 20px 0;
}

/* 等级面板 */
.tier-board {
  flex: 1;
  min-width: 0;
}

.tier-row {
  display: flex;
  margin-bottom: 4px;
  border-radius: var(--radius);
  overflow: hidden;
  border: 1px solid var(--border);
}

.tier-label {
  width: 100px;
  min-width: 100px;
  padding: 12px;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 6px;
  flex-shrink: 0;
}
.tier-name {
  font-size: 1.2rem;
  font-weight: 800;
  color: #fff;
  text-shadow: 0 1px 3px rgba(0,0,0,0.3);
  outline: none;
  text-align: center;
  min-width: 20px;
}
.tier-name:focus {
  background: rgba(255,255,255,0.15);
  border-radius: 4px;
  padding: 2px 6px;
}
.tier-label-actions {
  display: flex;
  align-items: center;
  gap: 4px;
}
.color-picker {
  width: 20px;
  height: 20px;
  border: none;
  background: none;
  cursor: pointer;
  border-radius: 4px;
  padding: 0;
}
.color-picker::-webkit-color-swatch-wrapper { padding: 0; }
.color-picker::-webkit-color-swatch { border: 2px solid rgba(255,255,255,0.4); border-radius: 4px; }

.tier-items {
  flex: 1;
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
  padding: 10px;
  min-height: 60px;
  background: var(--bg-card);
  align-items: flex-start;
  align-content: flex-start;
  transition: background 0.2s;
}
.tier-items.drag-over {
  background: var(--bg-hover);
  outline: 2px dashed var(--primary);
  outline-offset: -2px;
}

.tier-item {
  display: flex;
  align-items: center;
  gap: 6px;
  padding: 8px 14px;
  background: var(--bg);
  border: 1px solid var(--border);
  border-radius: var(--radius-sm);
  cursor: grab;
  transition: all 0.2s;
  font-size: 0.85rem;
  font-weight: 500;
}
.tier-item:hover {
  border-color: var(--primary);
  transform: translateY(-1px);
  box-shadow: 0 4px 12px rgba(0,0,0,0.3);
}
.tier-item:active { cursor: grabbing; }

.tier-empty {
  color: var(--text-dim);
  font-size: 0.8rem;
  padding: 8px;
  width: 100%;
  text-align: center;
}

/* 添加等级 */
.add-tier-row {
  padding: 12px;
  border: 2px dashed var(--border);
  border-radius: var(--radius);
  margin-top: 8px;
}
.add-tier-controls {
  display: flex;
  gap: 8px;
  align-items: center;
}
.add-tier-controls .input {
  width: 160px;
}

/* 预设 */
.presets {
  margin-top: 32px;
  padding: 24px 0;
  border-top: 1px solid var(--border);
}
.presets h3 {
  font-size: 1rem;
  font-weight: 700;
  margin-bottom: 12px;
}
.preset-buttons {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
}
.btn-preset {
  padding: 8px 16px;
  border-radius: 20px;
  border: 1px solid var(--border);
  background: var(--bg-card);
  color: var(--text);
  font-size: 0.82rem;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s;
  font-family: inherit;
}
.btn-preset:hover {
  border-color: var(--primary);
  color: var(--primary);
  background: rgba(108, 92, 231, 0.1);
}

/* Footer */
.footer {
  text-align: center;
  padding: 32px 0;
  color: var(--text-dim);
  font-size: 0.8rem;
  border-top: 1px solid var(--border);
  margin-top: 32px;
}

/* 响应式 */
@media (max-width: 768px) {
  .main {
    flex-direction: column;
  }
  .item-pool {
    width: 100%;
    position: static;
  }
  .pool-items {
    max-height: 200px;
  }
  .tier-label {
    width: 70px;
    min-width: 70px;
    padding: 8px;
  }
  .tier-name {
    font-size: 1rem;
  }
  .nav-actions .btn-outline {
    display: none;
  }
}
</style>
