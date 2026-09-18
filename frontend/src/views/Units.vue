<template>
  <div>
    <div class="toolbar">
      <div>
        <h2 class="page-title">探方 / 发掘单位</h2>
        <p class="page-sub">登记探方编号、深度与地层简述</p>
      </div>
      <button class="btn" @click="openCreate">新增探方</button>
    </div>

    <div class="card">
      <label style="max-width: 260px; margin-bottom: 1rem;">
        按工地筛选
        <select v-model="filterSiteId" @change="load">
          <option value="">全部工地</option>
          <option v-for="s in sites" :key="s.id" :value="String(s.id)">{{ s.name }}</option>
        </select>
      </label>
      <table class="table">
        <thead>
          <tr>
            <th>编号</th>
            <th>所属工地</th>
            <th class="depth-col">深度区间(m)</th>
            <th>地层简述</th>
            <th>操作</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="item in list" :key="item.id">
            <td>{{ item.code }}</td>
            <td>{{ item.site?.name || '-' }}</td>
            <td>
              <div class="depth-cell">
                <div class="depth-bar" :title="`${item.depthMin} ~ ${item.depthMax} m（本页最深 ${fmt(pageMaxDepth)} m）`">
                  <div class="depth-seg" :style="depthStyle(item)"></div>
                </div>
                <div class="depth-scale">
                  <span>0</span>
                  <span>{{ fmt(pageMaxDepth) }} m</span>
                </div>
                <div class="depth-text">{{ item.depthMin }} ~ {{ item.depthMax }} m</div>
              </div>
            </td>
            <td>{{ item.stratumDesc || '-' }}</td>
            <td>
              <button class="btn secondary small" @click="openEdit(item)">编辑</button>
              <button class="btn danger small" @click="remove(item)">删除</button>
            </td>
          </tr>
        </tbody>
      </table>
      <p v-if="!list.length" class="page-sub">暂无数据</p>
      <p v-if="error" class="error">{{ error }}</p>
    </div>

    <div v-if="showModal" class="modal-mask" @click.self="showModal = false">
      <div class="modal">
        <h3>{{ form.id ? '编辑探方' : '新增探方' }}</h3>
        <div class="form-grid">
          <label>
            所属工地
            <select v-model.number="form.siteId">
              <option :value="0" disabled>请选择</option>
              <option v-for="s in sites" :key="s.id" :value="s.id">{{ s.name }}</option>
            </select>
          </label>
          <label>
            编号
            <input v-model="form.code" placeholder="如 T1" />
          </label>
          <label>
            深度下限(m)
            <input
              v-model.number="form.depthMin"
              type="number"
              step="0.1"
              :class="{ invalid: depthError }"
            />
          </label>
          <label>
            深度上限(m)
            <input
              v-model.number="form.depthMax"
              type="number"
              step="0.1"
              :class="{ invalid: depthError }"
            />
          </label>
          <p v-if="depthError" class="error depth-hint full">{{ depthError }}</p>
          <p v-else class="depth-hint full depth-hint-ok">留空按 0 m 计；下限不得大于上限，深度单位为米。</p>
          <label class="full">
            地层简述
            <textarea v-model="form.stratumDesc" />
          </label>
        </div>
        <p v-if="formError" class="error">{{ formError }}</p>
        <div class="modal-actions">
          <button class="btn secondary" @click="showModal = false">取消</button>
          <button class="btn" :disabled="!!depthError" @click="save">保存</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { computed, onMounted, reactive, ref } from 'vue'
import api from '../api/http'

const list = ref([])
const sites = ref([])
const filterSiteId = ref('')
const error = ref('')
const formError = ref('')
const showModal = ref(false)
const form = reactive({
  id: null,
  siteId: 0,
  code: '',
  depthMin: 0,
  depthMax: 0,
  stratumDesc: ''
})

// 条形以当前筛选结果中的最大深度为满刻度，筛选工地后随之重算。
const pageMaxDepth = computed(() => {
  return list.value.reduce((max, u) => Math.max(max, Number(u.depthMax) || 0), 0)
})

function fmt(v) {
  return (Math.round((Number(v) || 0) * 100) / 100).toString()
}

function depthStyle(item) {
  const max = pageMaxDepth.value
  const min = Number(item.depthMin) || 0
  const top = Number(item.depthMax) || 0
  if (max <= 0) return { left: '0%', width: '0%' }
  const leftPct = Math.min(100, Math.max(0, (min / max) * 100))
  const widthPct = Math.min(100 - leftPct, Math.max(0, ((top - min) / max) * 100))
  return { left: `${leftPct}%`, width: `${widthPct}%` }
}

// 实时校验深度区间：留空补 0，非数字/负数/下限大于上限均立即提示。
const depthError = computed(() => {
  const parse = (raw) => (raw === '' || raw === null || raw === undefined ? 0 : Number(raw))
  const min = parse(form.depthMin)
  const max = parse(form.depthMax)
  if (!Number.isFinite(min) || !Number.isFinite(max)) return '深度需填写数字'
  if (min < 0 || max < 0) return '深度不能为负数（单位：米）'
  if (min > max) return `深度下限 ${fmt(min)} m 大于上限 ${fmt(max)} m，请调整区间`
  return ''
})

async function loadSites() {
  const { data } = await api.get('/sites')
  sites.value = data
}

async function load() {
  error.value = ''
  try {
    const params = {}
    if (filterSiteId.value) params.siteId = filterSiteId.value
    const { data } = await api.get('/units', { params })
    list.value = data
  } catch (e) {
    error.value = e.response?.data?.error || '加载失败'
  }
}

function openCreate() {
  Object.assign(form, {
    id: null,
    siteId: sites.value[0]?.id || 0,
    code: '',
    depthMin: 0,
    depthMax: 1,
    stratumDesc: ''
  })
  formError.value = ''
  showModal.value = true
}

function openEdit(item) {
  Object.assign(form, {
    id: item.id,
    siteId: item.siteId,
    code: item.code,
    depthMin: item.depthMin,
    depthMax: item.depthMax,
    stratumDesc: item.stratumDesc
  })
  formError.value = ''
  showModal.value = true
}

async function save() {
  if (depthError.value) return
  formError.value = ''
  const numOrZero = (v) => {
    const n = v === '' || v === null || v === undefined ? 0 : Number(v)
    return Number.isFinite(n) ? n : 0
  }
  try {
    const payload = {
      siteId: form.siteId,
      code: form.code,
      depthMin: numOrZero(form.depthMin),
      depthMax: numOrZero(form.depthMax),
      stratumDesc: form.stratumDesc
    }
    if (form.id) {
      await api.put(`/units/${form.id}`, payload)
    } else {
      await api.post('/units', payload)
    }
    showModal.value = false
    await load()
  } catch (e) {
    formError.value = e.response?.data?.error || '保存失败'
  }
}

async function remove(item) {
  if (!confirm(`确认删除探方「${item.code}」？`)) return
  try {
    await api.delete(`/units/${item.id}`)
    await load()
  } catch (e) {
    alert(e.response?.data?.error || '删除失败')
  }
}

onMounted(async () => {
  await loadSites()
  await load()
})
</script>

<style scoped>
.depth-col {
  min-width: 200px;
}

.depth-cell {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
  min-width: 180px;
}

.depth-bar {
  position: relative;
  height: 10px;
  border-radius: 999px;
  background: #ece2d1;
  overflow: hidden;
}

.depth-seg {
  position: absolute;
  top: 0;
  bottom: 0;
  border-radius: 999px;
  background: linear-gradient(90deg, #b07a45, var(--accent));
}

.depth-scale {
  display: flex;
  justify-content: space-between;
  font-size: 0.72rem;
  color: var(--muted);
}

.depth-text {
  font-size: 0.8rem;
  color: var(--muted);
}

.depth-hint {
  margin: 0;
  font-size: 0.82rem;
}

.depth-hint-ok {
  color: var(--muted);
}

input.invalid {
  border-color: var(--danger);
}

.btn:disabled {
  opacity: 0.55;
  cursor: not-allowed;
}
</style>
