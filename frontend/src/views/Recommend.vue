<template>
  <div class="recommend-container">

    <div class="search-header">
      <div class="search-box-wrapper">
        <div class="primary-search">
          <el-select
              v-model="queryForm.city"
              placeholder="城市"
              filterable
              clearable
              class="city-select"
              size="large"
          >
            <template #prefix><el-icon><Location /></el-icon></template>
            <el-option v-for="city in cityOptions" :key="city" :label="city" :value="city"/>
          </el-select>

          <div class="divider"></div>

          <el-input
              v-model="queryForm.keyword"
              placeholder="搜索职位 (如: Python, 产品经理)"
              class="main-input"
              size="large"
              clearable
              @keyup.enter="handleSearch"
              @clear="searchedKeyword = ''"
          >
            <template #prefix><el-icon><Search /></el-icon></template>
          </el-input>

          <el-button type="primary" size="large" class="search-btn" @click="handleSearch" :loading="loading">
            搜职位
          </el-button>
        </div>

        <div class="secondary-filters">
          <el-select v-model="queryForm.experience" placeholder="经验要求" size="default" clearable class="filter-item" @change="handleFilterApply">
            <template #prefix><el-icon><Timer /></el-icon></template>
            <el-option label="应届生" value="应届生" />
            <el-option label="1-3年" value="1-3年" />
            <el-option label="3-5年" value="3-5年" />
            <el-option label="5年以上" value="5-10年" />
          </el-select>

          <el-select v-model="queryForm.education" placeholder="学历要求" size="default" clearable class="filter-item" @change="handleFilterApply">
            <template #prefix><el-icon><Reading /></el-icon></template>
            <el-option label="本科" value="本科" />
            <el-option label="硕士" value="硕士" />
            <el-option label="大专" value="大专" />
          </el-select>

          <el-select v-model="queryForm.salary" placeholder="最低薪资" size="default" clearable class="filter-item" @change="handleFilterApply">
            <template #prefix><el-icon><Money /></el-icon></template>
            <el-option label="3k+" :value="3000" />
            <el-option label="5k+" :value="5000" />
            <el-option label="10k+" :value="10000" />
            <el-option label="15k+" :value="15000" />
            <el-option label="20k+" :value="20000" />
            <el-option label="30k+" :value="30000" />
            <el-option label="50k+" :value="50000" />
            <el-option label="100k+" :value="100000" />
          </el-select>

          <el-select
              v-model="queryForm.skill"
              multiple
              collapse-tags
              collapse-tags-tooltip
              filterable
              reserve-keyword
              placeholder="搜技能"
              size="default"
              class="filter-item skill-select"
              @change="handleFilterApply"
          >
            <el-option v-for="item in skillOptions" :key="item" :label="item" :value="item"/>
          </el-select>

          <span class="result-tip" v-if="jobList.length > 0">
             找到 <span class="highlight">{{ jobList.length }}</span> 个好机会
          </span>
        </div>
      </div>
    </div>

    <div class="result-area">
      <el-empty v-if="jobList.length === 0 && !loading" :image-size="160" description="暂无数据">
        <template #description>
          <p class="empty-text">没找到心仪的岗位？</p>
        </template>
        <div v-if="searchedKeyword || queryForm.keyword">
          <el-button type="primary" size="default" :icon="RefreshRight" class="crawl-btn" @click="handleQuickCrawl" :disabled="realtimeStatus.is_running" :loading="isCrawling">
            {{ realtimeStatus.is_running ? '全网采集运行中...' : `立即采集 "${searchedKeyword || queryForm.keyword}"` }}
          </el-button>
        </div>
      </el-empty>

      <el-row :gutter="15" v-if="jobList.length > 0">
        <el-col :span="8" v-for="job in pagedJobList" :key="job.id">
          <el-card shadow="hover" class="job-card" @click="goToDetail(job.detail_url)">
            <div class="card-top">
              <div class="job-name-box">
                <h3 class="job-name" v-html="highlightKeyword(job.job_name)"></h3>
                <span class="salary-text">{{ job.salary }}</span>
              </div>
              <el-tag type="danger" effect="plain" size="small" class="score-badge">推荐分:{{ job.score }}</el-tag>
            </div>
            <div class="card-mid">
              <div class="company-row">
                <span class="company-name">{{ job.company || '未知公司' }}</span>
              </div>
              <div class="info-row">
                <span>{{ job.city ? job.city.split('-')[0] : '未知' }}</span>
                <el-divider direction="vertical" />
                <span>{{ job.experience || '经验不限' }}</span>
                <el-divider direction="vertical" />
                <span>{{ job.education }}</span>
              </div>
            </div>
            <div class="card-bot">
              <div class="tag-scroll">
                <el-tag v-for="tag in parseSkills(job.skills).slice(0, 4)" :key="tag" type="info" size="small" class="skill-tag">{{ tag }}</el-tag>
              </div>
            </div>
          </el-card>
        </el-col>
      </el-row>

      <div class="pagination-box" v-if="jobList.length > 0">
        <el-pagination background layout="prev, pager, next" :total="jobList.length" :page-size="pageSize" v-model:current-page="currentPage" @current-change="handlePageChange"/>

        <div v-if="isLastPage" class="load-more-text">
          <el-button link type="primary" @click="handleQuickCrawl" :disabled="realtimeStatus.is_running">
            {{ realtimeStatus.is_running ? '后台采集中...' : '没看够？点此采集更多' }}
          </el-button>
        </div>
      </div>
    </div>

    <transition name="el-zoom-in-bottom">
      <div v-if="realtimeStatus.is_running" class="spider-status-wrapper">
        <div class="spider-status-card">
          <div class="status-left">
            <el-spinner class="spinner" /> <span class="status-title">采集引擎运行中</span>
          </div>
          <div class="status-center">
            {{ realtimeStatus.log }}
          </div>
          <div class="status-right">
            已入库: {{ realtimeStatus.total_added }}
          </div>
        </div>
      </div>
    </transition>

  </div>
</template>

<script setup>
import {ref, reactive, onMounted, computed, watch, onUnmounted} from "vue";
import axios from "axios";
import { ElMessage } from "element-plus";
import { Location, Timer, Reading, Search, RefreshRight, Money } from '@element-plus/icons-vue'

// =================== 状态定义 ===================
const loading = ref(false)
const isCrawling = ref(false)
const jobList = ref([])
const cityOptions = ref([])
const skillOptions = ref([])
const searchedKeyword = ref('')
const currentPage = ref(1)
const pageSize = ref(12) // 改成12个，3列布局更好看

// 爬虫实时状态
const realtimeStatus = reactive({
  is_running: false,
  total_added: 0,
  log: ''
})

const queryForm = reactive({
  keyword: '',
  city: '',
  skill: [],
  salary: null, // 改成 null
  education: '',
  experience: ''
});

// =================== 计算属性 ===================
const pagedJobList = computed(() => {
  const start = (currentPage.value - 1) * pageSize.value
  const end = start + pageSize.value
  return jobList.value.slice(start, end)
})

const isLastPage = computed(() => {
  if (jobList.value.length === 0) return false
  const maxPage = Math.ceil(jobList.value.length / pageSize.value)
  return currentPage.value === maxPage
})

watch(() => jobList.value, () => { currentPage.value = 1 })

// =================== 核心功能 ===================

const handleSearch = () => {
  queryForm.skill = []
  searchedKeyword.value = queryForm.keyword
  getRecommendations(true)
}

const handleFilterApply = () => { getRecommendations(false) }

const handlePageChange = () => {
  document.querySelector('.result-area')?.scrollIntoView({ behavior: 'smooth' })
}

const getRecommendations = async (isNewSearch = false, silent = false) => {
  if (!queryForm.keyword && isNewSearch) {
    if(!silent) ElMessage.warning('请输入搜索关键词')
    return
  }
  if (!silent) loading.value = true
  try {
    const res = await axios.get('http://127.0.0.1:5000/api/recommend', { params: queryForm })
    if (res.data.code === 200) {
      jobList.value = res.data.data
      if (isNewSearch) {
        updateSkillOptions(jobList.value)
        searchedKeyword.value = queryForm.keyword
      }
    }
  } catch (error){
    if (!silent) ElMessage.error('服务不可用')
  } finally {
    if (!silent) loading.value = false
  }
}

const handleQuickCrawl = async () => {
  const keyword = searchedKeyword.value || queryForm.keyword
  const city = queryForm.city || ''

  if (!keyword) return ElMessage.warning("请先输入关键词")
  if (realtimeStatus.is_running) return;

  isCrawling.value = true
  try {
    const res = await axios.post('http://127.0.0.1:5000/api/spider/start', { keyword: keyword, city: city })
    if (res.data.code === 200) {
      ElMessage.success('采集任务已启动')
      lastTotalAdded = 0
      startStatusPolling()
    } else {
      ElMessage.warning(res.data.msg)
      isCrawling.value = false
    }
  } catch (error) {
    ElMessage.error('无法启动采集')
    isCrawling.value = false
  }
}

// 轮询状态
let statusTimer = null
let lastTotalAdded = 0

const startStatusPolling = () => {
  if (statusTimer) clearInterval(statusTimer)
  statusTimer = setInterval(async () => {
    try {
      const res = await axios.get('http://127.0.0.1:5000/api/spider/status')
      if (res.data.code === 200) {
        const data = res.data.data
        realtimeStatus.is_running = data.is_running
        realtimeStatus.log = data.log // 更新日志

        if (data.total_added > lastTotalAdded) {
          lastTotalAdded = data.total_added
          realtimeStatus.total_added = data.total_added
          getRecommendations(false, true)
        }
        if (!data.is_running) {
          clearInterval(statusTimer)
          isCrawling.value = false
          getRecommendations(false, true)
        }
      }
    } catch (e) {}
  }, 1000)
}

onUnmounted(() => { if (statusTimer) clearInterval(statusTimer) })

// 工具函数
const parseSkills = (s) => s ? s.replace(/ /g, '').split(',').slice(0, 6) : []
const highlightKeyword = (t) => {
  if (!queryForm.keyword) return t
  const reg = new RegExp(queryForm.keyword, 'gi')
  return t.replace(reg, m => `<span style="color:#f56c6c">${m}</span>`)
}
const updateSkillOptions = (jobs) => {
  const s = new Set(); jobs.forEach(j => j.skills && j.skills.split(',').forEach(t => t.trim() && t.length<15 && s.add(t.trim())))
  skillOptions.value = Array.from(s).sort()
}
const goToDetail = (url) => url && window.open(url, '_blank')
const loadOptions = async () => { try { const r = await axios.get('http://127.0.0.1:5000/api/cities'); if(r.data.code===200) cityOptions.value=r.data.data }catch(e){} }
onMounted(loadOptions)
</script>

<style scoped>
.recommend-container {
  max-width: 1200px;
  margin: 0 auto;
  padding-bottom: 80px;
}

/* 🔥 1. 搜索头部的优化样式 */
.search-header {
  background: #fff;
  border-radius: 12px;
  box-shadow: 0 4px 20px rgba(0,0,0,0.05);
  padding: 20px 25px;
  margin-bottom: 20px;
  border: 1px solid #ebeef5;
}

.primary-search {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-bottom: 15px;
}

/* 城市选择器 */
.city-select { width: 140px; }
:deep(.city-select .el-input__wrapper) {
  border-radius: 8px 0 0 8px; /* 左圆角 */
  box-shadow: none !important;
  border: 1px solid #dcdfe6;
}

.divider {
  width: 1px;
  height: 24px;
  background: #dcdfe6;
  margin: 0 -11px; /* 视觉上的分割线 */
  z-index: 2;
}

/* 搜索主输入框 */
.main-input { flex: 1; }
:deep(.main-input .el-input__wrapper) {
  border-radius: 0; /* 直角 */
  box-shadow: none !important;
  border: 1px solid #dcdfe6;
  border-left: none;
}

/* 搜索按钮 */
.search-btn {
  border-radius: 0 8px 8px 0; /* 右圆角 */
  padding: 0 30px;
  font-weight: bold;
  font-size: 16px;
}

/* 次级筛选行 */
.secondary-filters {
  display: flex;
  align-items: center;
  gap: 12px;
  flex-wrap: wrap;
}

.filter-item { width: 110px; }
.skill-select { width: 160px; }
:deep(.filter-item .el-input__wrapper) {
  background-color: #f5f7fa;
  box-shadow: none !important;
  border-radius: 16px; /* 胶囊形状 */
  border: 1px solid transparent;
  transition: all 0.3s;
}
:deep(.filter-item .el-input__wrapper:hover) {
  background-color: #fff;
  border-color: #c0c4cc;
}

.result-tip {
  margin-left: auto;
  font-size: 13px;
  color: #909399;
}
.highlight { color: #409eff; font-weight: bold; }


/* 🔥 2. 职位卡片优化 */
.result-area { min-height: 400px; }
.empty-text { color: #909399; margin-bottom: 15px; }

.job-card {
  margin-bottom: 15px;
  cursor: pointer;
  border-radius: 10px;
  border: 1px solid #f0f2f5;
  transition: transform 0.2s, box-shadow 0.2s;
  height: 170px; /* 更紧凑的高度 */
  display: flex;
  flex-direction: column;
}
.job-card:hover {
  transform: translateY(-3px);
  box-shadow: 0 8px 16px rgba(0,0,0,0.08);
  border-color: #dcdfe6;
}
:deep(.el-card__body) { padding: 15px; height: 100%; box-sizing: border-box; display: flex; flex-direction: column; justify-content: space-between; }

.card-top { display: flex; justify-content: space-between; align-items: flex-start; }
.job-name { margin: 0; font-size: 15px; font-weight: bold; color: #303133; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; max-width: 80%; }
.salary-text { color: #f56c6c; font-weight: 800; font-size: 14px; margin-top: 4px; display: block; }

.card-mid { font-size: 12px; color: #606266; margin: 8px 0; }
.company-name { font-weight: 500; display: block; margin-bottom: 4px; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.info-row { color: #909399; }

.card-bot { margin-top: auto; }
.tag-scroll { display: flex; gap: 6px; overflow: hidden; }
.skill-tag { border: none; background: #f4f4f5; color: #909399; }

/* 分页 */
.pagination-box { display: flex; flex-direction: column; align-items: center; margin-top: 20px; }
.load-more-text { margin-top: 10px; }

/* 悬浮条样式 (保持不变) */
.spider-status-wrapper {
  position: fixed; bottom: 25px; left: 200px; right: 0;
  display: flex; justify-content: center; z-index: 2000; pointer-events: none;
}
.spider-status-card {
  background: rgba(33, 33, 33, 0.95); backdrop-filter: blur(10px); color: #fff;
  padding: 10px 20px; border-radius: 50px; box-shadow: 0 10px 30px rgba(0,0,0,0.2);
  display: flex; align-items: center; gap: 15px; pointer-events: auto;
  font-size: 13px; min-width: 350px;
}
.status-left { display: flex; align-items: center; gap: 8px; color: #409eff; font-weight: bold; }
.spinner { width: 14px; height: 14px; border: 2px solid #409eff; border-top-color: transparent; border-radius: 50%; animation: spin 1s linear infinite; }
.status-center { flex: 1; text-align: center; color: #ccc; max-width: 200px; white-space: nowrap; overflow: hidden; text-overflow: ellipsis;}
.status-right { background: #67c23a; padding: 1px 8px; border-radius: 10px; font-size: 12px; }
@keyframes spin { from { transform: rotate(0deg); } to { transform: rotate(360deg); } }
</style>