<template>
  <div class="app" :data-theme="theme">
    <header class="topbar">
      <div class="topbar-left">
        <button class="menu-toggle" @click="toggleSidebar">☰</button>
        <div class="logo"><span>ME</span>模块英语</div>
      </div>
      <div class="controls">
        <button @click="prev">◀</button>
        <span class="page-indicator">{{ currentIndex + 1 }} / {{ filteredSentences.length }}</span>
        <button @click="next">▶</button>
        <button @click="toggleCN">{{ isCNHidden ? '显示中文' : '隐藏中文' }}</button>
        <button @click="toggleEN">{{ isENHidden ? '显示英文' : '隐藏英文' }}</button>
        <button @click="openWordbook">📖 生词本</button>
        <button @click="speakFull">🔊 整句朗读</button>
        <button @click="startFullTest">✏️ 整句自测</button>
        <select v-model="speed">
          <option value="0.7">0.7x</option>
          <option value="1">1x</option>
          <option value="1.3">1.3x</option>
        </select>
        <button @click="cycleTheme">🎨 主题</button>
      </div>
    </header>

    <div class="main-layout" @click="closeSidebarOnClickOutside">
      <aside class="sidebar" :class="{ 'sidebar-visible': sidebarVisible }">
        <div class="sidebar-inner">
          <div class="sidebar-section">
            <div class="sidebar-title" @click="toggleSection('difficulty')">
              <span>📚 难度</span>
              <span class="toggle-icon">{{ expandedSections.difficulty ? '▼' : '▶' }}</span>
            </div>
            <div v-if="expandedSections.difficulty" class="sidebar-options">
              <button class="sidebar-btn" @click="filterLevel('all')">📖 全部</button>
              <button class="sidebar-btn" @click="filterLevel('beginner')">🟢 初级</button>
              <button class="sidebar-btn" @click="filterLevel('intermediate')">🟡 中级</button>
              <button class="sidebar-btn" @click="filterLevel('advanced')">🔴 高级</button>
            </div>
          </div>
          <div class="sidebar-section">
            <div class="sidebar-title" @click="toggleSection('video')">
              <span>🎬 视频</span>
              <span class="toggle-icon">{{ expandedSections.video ? '▼' : '▶' }}</span>
            </div>
            <div v-if="expandedSections.video" class="sidebar-options">
              <button class="sidebar-btn" @click="showSyntax = true">📖 句法</button>
              <button class="sidebar-btn" @click="showGrammar = true">📚 词法</button>
            </div>
          </div>
          <div class="sidebar-section">
            <div class="sidebar-title" @click="toggleSection('tool')">
              <span>🔧 工具</span>
              <span class="toggle-icon">{{ expandedSections.tool ? '▼' : '▶' }}</span>
            </div>
            <div v-if="expandedSections.tool" class="sidebar-options">
              <button class="sidebar-btn" @click="showParser = true">🧩 复杂句拆解器</button>
              <button class="sidebar-btn" @click="showArticleInput = true">📝 文章输入学习</button>
            </div>
          </div>
        </div>
      </aside>

      <div class="main-container">
        <main class="main-area" ref="mainAreaRef" @touchstart="onTouchStart" @touchend="onTouchEnd">
          <div v-if="loading" class="loading">加载句子中...</div>
          <div v-else-if="filteredSentences.length === 0" class="loading">暂无句子，请检查数据库</div>
          <div v-else class="sentence-card">
            <div class="sentence-english">{{ currentSentence.english }}</div>
            <div class="chinese-translation" :class="{ 'hidden-cn': isCNHidden }" @dblclick="editTranslation">
              {{ currentSentence.chinese }}
            </div>
            <div class="modules-container" v-if="currentModules && currentModules.length > 0">
              <div
                v-for="(mod, modIdx) in currentModules"
                :key="modIdx"
                class="module-row"
                :style="{ backgroundColor: `${mod.color}20`, borderLeftColor: mod.color }"
                :class="{ 'module-highlight': highlightedModule === mod }"
                @click.stop="handleModuleClick(mod)"
              >
                <span class="module-label" :style="{ color: mod.color, background: `${mod.color}15` }">
                  🏷️ {{ mod.label }}
                </span>
                <div class="module-phrases">
                  <span
                    v-for="(phrase, phIdx) in mod.phrases"
                    :key="phIdx"
                    class="phrase-item"
                    :style="{ backgroundColor: `${mod.color}35` }"
                    :class="{ 'phrase-highlight': highlightedPhrase === phrase }"
                    @click.stop="handlePhraseClick(phrase)"
                  >
                    <span
                      v-for="(word, wIdx) in phrase.words"
                      :key="wIdx"
                      class="word"
                      :class="{
                        'hidden-en': isENHidden,
                        'word-highlight': highlightedWord === word
                      }"
                      @click.stop="handleWordClick($event, word)"
                    >
                      {{ word.w }}{{ wIdx < phrase.words.length - 1 ? ' ' : '' }}
                    </span>
                    <span class="phrase-cn" :class="{ 'hidden-cn-text': isCNHidden }">
                      {{ phrase.cn }}
                    </span>
                    <span class="phrase-label">{{ phrase.label }}</span>
                    <button class="test-btn" @click.stop="testPhrase(phrase)">✏️</button>
                  </span>
                </div>
              </div>
            </div>
          </div>
        </main>
      </div>
    </div>

    <!-- 单词弹窗 -->
    <Teleport to="body">
      <div v-if="tooltipVisible" class="word-tooltip" :style="{ top: tooltipY + 'px', left: tooltipX + 'px' }">
        <div class="tooltip-word">{{ tooltipWord.w }}</div>
        <div class="tooltip-pos">{{ tooltipWord.pos || '单词' }}</div>
        <div class="tooltip-meaning">{{ tooltipWord.m || tooltipWord.w }}</div>
      </div>
    </Teleport>

    <!-- 自测弹窗 -->
    <div v-if="testMode" class="test-modal-overlay" @click.self="testMode = false">
      <div class="test-modal">
        <div class="test-modal-header">
          <h3>✏️ 自测练习</h3>
          <button class="test-modal-close" @click="closeTest">✕</button>
        </div>
        <div class="test-modal-body">
          <div class="test-prompt-card">
            <span class="test-prompt-icon">🔤</span>
            <span class="test-prompt-text">{{ testPrompt }}</span>
          </div>
          <div v-if="toastMessage" class="toast-message" :class="toastType">{{ toastMessage }}</div>
          <input ref="testInputRef" type="text" class="test-input" v-model="testInput" @keyup.enter="submitTest" placeholder="输入答案..." />
          <div class="test-buttons">
            <button class="test-submit" @click="submitTest">✅ 提交</button>
            <button class="test-cancel" @click="closeTest">取消</button>
          </div>
        </div>
      </div>
    </div>

    <!-- 生词本弹窗 -->
    <div v-if="showWordbook" class="modal-overlay" @click.self="showWordbook = false">
      <div class="modal modal-large">
        <h3>📖 生词本</h3>
        <div class="book-tabs">
          <button :class="{ active: bookTab === 'words' }" @click="switchTab('words')">单词 ({{ wordbook.length }})</button>
          <button :class="{ active: bookTab === 'phrases' }" @click="switchTab('phrases')">短语 ({{ collectedPhrases.length }})</button>
        </div>
        <div class="book-header">
          <label class="select-all-label">
            <input type="checkbox" v-model="selectAllWords" @change="toggleSelectAllWords" v-if="bookTab === 'words'" />
            <input type="checkbox" v-model="selectAllPhrases" @change="toggleSelectAllPhrases" v-if="bookTab === 'phrases'" />
            全选
          </label>
          <button class="delete-selected" @click="deleteSelectedItems" :disabled="selectedCount === 0">
            🗑️ 删除选中 ({{ selectedCount }})
          </button>
        </div>
        <div class="book-list">
          <div v-if="bookTab === 'words'">
            <div v-if="wordbook.length === 0" class="empty-tip">暂无收藏单词</div>
            <div v-for="(item, idx) in wordbook" :key="idx" class="book-item">
              <input type="checkbox" :value="idx" v-model="selectedWordIndices" class="book-checkbox" />
              <span class="book-num">{{ idx + 1 }}.</span>
              <span class="book-content">{{ item.word }} - {{ item.meaning || item.word }}</span>
              <button class="book-study" @click="startStudyMode('word', idx)">📖</button>
              <button class="book-spell" @click="startSpellMode('word', idx)">✏️</button>
              <button class="book-delete" @click="removeWord(idx)">🗑</button>
            </div>
          </div>
          <div v-if="bookTab === 'phrases'">
            <div v-if="collectedPhrases.length === 0" class="empty-tip">暂无收藏短语</div>
            <div v-for="(item, idx) in collectedPhrases" :key="idx" class="book-item">
              <input type="checkbox" :value="idx" v-model="selectedPhraseIndices" class="book-checkbox" />
              <span class="book-num">{{ idx + 1 }}.</span>
              <span class="book-content">{{ item.en }} - {{ item.cn }}</span>
              <button class="book-study" @click="startStudyMode('phrase', idx)">📖</button>
              <button class="book-spell" @click="startSpellMode('phrase', idx)">✏️</button>
              <button class="book-delete" @click="removePhrase(idx)">🗑</button>
            </div>
          </div>
        </div>
        <button class="close-modal" @click="showWordbook = false">关闭</button>
      </div>
    </div>

    <!-- 学习模式 -->
    <div v-if="studyMode" class="modal-overlay" @click.self="studyMode = false">
      <div class="study-modal">
        <div class="study-modal-header">
          <h3>📖 学习模式</h3>
          <button class="study-modal-close" @click="studyMode = false">✕</button>
        </div>
        <div class="study-card">
          <div class="study-card-word">{{ studyCardItem.word || studyCardItem.en }}</div>
          <div class="study-card-meaning">{{ studyCardItem.meaning || studyCardItem.cn }}</div>
          <button class="study-card-speak" @click="speak(studyCardItem.word || studyCardItem.en)">🔊</button>
        </div>
        <div class="study-nav">
          <button @click="prevStudyCard" :disabled="studyCardIndex === 0">◀</button>
          <span>{{ studyCardIndex + 1 }} / {{ studyCardList.length }}</span>
          <button @click="nextStudyCard" :disabled="studyCardIndex === studyCardList.length - 1">▶</button>
        </div>
        <button class="study-exit" @click="studyMode = false">退出</button>
      </div>
    </div>

    <!-- 拼写模式 -->
    <div v-if="spellMode" class="modal-overlay" @click.self="spellMode = false">
      <div class="spell-modal">
        <div class="spell-modal-header">
          <h3>✏️ 拼写模式</h3>
          <button class="spell-modal-close" @click="spellMode = false">✕</button>
        </div>
        <div class="spell-card">
          <div class="spell-card-hint">{{ spellCardItem.meaning || spellCardItem.cn }}</div>
          <input ref="spellInputRef" type="text" class="spell-input" v-model="spellInput" @keyup.enter="checkSpell" placeholder="输入英文..." />
          <div v-if="spellFeedback" class="spell-feedback" :class="spellFeedbackType">{{ spellFeedback }}</div>
          <div class="spell-buttons">
            <button class="spell-check" @click="checkSpell">✅ 检查</button>
            <button class="spell-skip" @click="nextSpellCard">跳过</button>
          </div>
        </div>
        <div class="spell-nav">
          <button @click="prevSpellCard" :disabled="spellCardIndex === 0">◀</button>
          <span>{{ spellCardIndex + 1 }} / {{ spellCardList.length }}</span>
          <button @click="nextSpellCard" :disabled="spellCardIndex === spellCardList.length - 1">▶</button>
        </div>
        <button class="spell-exit" @click="spellMode = false">退出</button>
      </div>
    </div>

    <!-- 占位弹窗 -->
    <div v-if="showParser" class="modal-overlay" @click.self="showParser = false">
      <div class="modal"><h3>🧩 复杂句拆解器</h3><p>功能开发中...</p><button @click="showParser = false">关闭</button></div>
    </div>
    <div v-if="showArticleInput" class="modal-overlay" @click.self="showArticleInput = false">
      <div class="modal"><h3>📝 文章输入学习</h3><p>功能开发中...</p><button @click="showArticleInput = false">关闭</button></div>
    </div>
    <div v-if="showSyntax" class="modal-overlay" @click.self="showSyntax = false">
      <div class="modal"><h3>📖 句法视频</h3><p>功能开发中...</p><button @click="showSyntax = false">关闭</button></div>
    </div>
    <div v-if="showGrammar" class="modal-overlay" @click.self="showGrammar = false">
      <div class="modal"><h3>📚 词法视频</h3><p>功能开发中...</p><button @click="showGrammar = false">关闭</button></div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, nextTick, watch, onUnmounted } from 'vue'
import { supabase } from './lib/supabase.js'

// ---------- 数据 ----------
const allSentences = ref([])
const filteredSentences = ref([])
const currentIndex = ref(0)
const loading = ref(true)

// 高亮
const highlightedWord = ref(null)
const highlightedPhrase = ref(null)
const highlightedModule = ref(null)

// 单词弹窗
const tooltipVisible = ref(false)
const tooltipWord = ref({})
const tooltipX = ref(0)
const tooltipY = ref(0)

const theme = ref('light')
const isCNHidden = ref(false)
const isENHidden = ref(false)
const speed = ref(1)

const currentLevel = ref('all')
const sidebarVisible = ref(false)
const expandedSections = ref({ difficulty: true, video: true, tool: true })

const showParser = ref(false)
const showArticleInput = ref(false)
const showSyntax = ref(false)
const showGrammar = ref(false)

// 自测
const testMode = ref(false)
const testItems = ref([])
const testItemIndex = ref(0)
const testInput = ref('')
const testPrompt = ref('')
const testInputRef = ref(null)
const toastMessage = ref('')
const toastType = ref('')
let toastTimerId = null
let consecutiveCorrect = 0

// 生词本
const showWordbook = ref(false)
const bookTab = ref('words')
const wordbook = ref([])
const collectedPhrases = ref([])
const selectedWordIndices = ref([])
const selectedPhraseIndices = ref([])
const selectAllWords = ref(false)
const selectAllPhrases = ref(false)

// 学习/拼写
const studyMode = ref(false)
const studyCardList = ref([])
const studyCardIndex = ref(0)
const studyCardItem = computed(() => studyCardList.value[studyCardIndex.value] || {})
const spellMode = ref(false)
const spellCardList = ref([])
const spellCardIndex = ref(0)
const spellInput = ref('')
const spellFeedback = ref('')
const spellFeedbackType = ref('')
const spellInputRef = ref(null)
const spellCardItem = computed(() => spellCardList.value[spellCardIndex.value] || {})

const touchStartX = ref(0)
const mainAreaRef = ref(null)

const selectedCount = computed(() => {
  if (bookTab.value === 'words') return selectedWordIndices.value.length
  return selectedPhraseIndices.value.length
})

// ---------- 语音优化（本地离线语音 + 预热 + 缓存）----------
let cachedUtterance = null
let cachedVoice = null
let speechWarmedUp = false

function getBestVoice() {
  if (cachedVoice) return cachedVoice
  
  const voices = window.speechSynthesis.getVoices()
  // 按优先级选择本地英语语音
  const preferredNames = ['Microsoft', 'Samantha', 'David', 'Zira', 'Alex', 'Siri']
  
  // 优先选择本地离线语音
  let bestVoice = voices.find(v => 
    v.lang === 'en-US' && 
    preferredNames.some(name => v.name.includes(name))
  )
  
  // 降级：任何英语语音
  if (!bestVoice) bestVoice = voices.find(v => v.lang === 'en-US')
  
  cachedVoice = bestVoice
  return cachedVoice
}

// 预热语音引擎（静默唤醒）
function warmupSpeech() {
  if (speechWarmedUp) return
  try {
    // 获取语音列表（触发引擎初始化）
    const voices = window.speechSynthesis.getVoices()
    getBestVoice()
    
    // 静默朗读一个空字符串
    const utterance = new SpeechSynthesisUtterance('')
    utterance.volume = 0
    window.speechSynthesis.speak(utterance)
    
    speechWarmedUp = true
    console.log('语音引擎已预热，可用语音数量：', voices.length)
  } catch(e) { console.warn('语音预热失败', e) }
}

function speak(text) {
  if (!text) return
  if (window.speechSynthesis.speaking) window.speechSynthesis.cancel()
  
  if (!cachedUtterance) {
    cachedUtterance = new SpeechSynthesisUtterance()
    cachedUtterance.lang = 'en-US'
  }
  
  cachedUtterance.text = text
  cachedUtterance.rate = speed.value
  cachedUtterance.voice = getBestVoice()
  
  window.speechSynthesis.speak(cachedUtterance)
}

function speakFull() { speak(currentSentence.value.english) }

// ---------- 辅助 ----------
function showToast(msg, type = 'info') {
  if (toastTimerId) clearTimeout(toastTimerId)
  toastMessage.value = msg
  toastType.value = type
  toastTimerId = setTimeout(() => { toastMessage.value = '' }, 1800)
}

function playSound(isCorrect) {
  try {
    const audioContext = new (window.AudioContext || window.webkitAudioContext)()
    const oscillator = audioContext.createOscillator()
    const gainNode = audioContext.createGain()
    oscillator.connect(gainNode)
    gainNode.connect(audioContext.destination)
    oscillator.type = 'sine'
    oscillator.frequency.value = isCorrect ? 880 : 440
    gainNode.gain.value = 0.25
    oscillator.start()
    gainNode.gain.exponentialRampToValueAtTime(0.00001, audioContext.currentTime + 0.4)
    oscillator.stop(audioContext.currentTime + 0.4)
  } catch(e) { console.warn('音效失败', e) }
}

function filterLevel(level) {
  currentLevel.value = level
  filteredSentences.value = level === 'all' ? [...allSentences.value] : allSentences.value.filter(s => s.level === level)
  currentIndex.value = 0
}

function prev() {
  if (filteredSentences.value.length === 0) return
  currentIndex.value = (currentIndex.value - 1 + filteredSentences.value.length) % filteredSentences.value.length
  clearAllHighlights()
}

function next() {
  if (filteredSentences.value.length === 0) return
  currentIndex.value = (currentIndex.value + 1) % filteredSentences.value.length
  clearAllHighlights()
}

function toggleSidebar() { sidebarVisible.value = !sidebarVisible.value }
function closeSidebarOnClickOutside(e) {
  const sidebar = document.querySelector('.sidebar')
  if (sidebarVisible.value && sidebar && !sidebar.contains(e.target) && !e.target.closest('.menu-toggle')) {
    sidebarVisible.value = false
  }
}
function toggleSection(section) { expandedSections.value[section] = !expandedSections.value[section] }
function handleKeydown(e) { if (e.key === 'ArrowLeft') prev(); if (e.key === 'ArrowRight') next() }
function onTouchStart(e) { touchStartX.value = e.touches[0].clientX }
function onTouchEnd(e) {
  const deltaX = e.changedTouches[0].clientX - touchStartX.value
  if (Math.abs(deltaX) > 50) deltaX > 0 ? prev() : next()
}

// 预加载句子音频（静默）
function preloadSentenceAudio(sentence) {
  if (!sentence?.modules) return
  sentence.modules.forEach(mod => mod.phrases.forEach(phrase => {
    const utterance = new SpeechSynthesisUtterance(phrase.en)
    utterance.volume = 0
    utterance.rate = 0.8
    window.speechSynthesis.speak(utterance)
  }))
}

function preloadNearbySentences() {
  const idx = currentIndex.value
  const start = Math.max(0, idx - 3)
  const end = Math.min(filteredSentences.value.length, idx + 4)
  for (let i = start; i < end; i++) preloadSentenceAudio(filteredSentences.value[i])
}

// 高亮与弹窗
function clearAllHighlights() {
  highlightedWord.value = null
  highlightedPhrase.value = null
  highlightedModule.value = null
  tooltipVisible.value = false
}
function handleModuleClick(mod) {
  clearAllHighlights()
  highlightedModule.value = mod
  const text = mod.phrases.map(p => p.en).join(' ')
  speak(text)
}
function handlePhraseClick(phrase) {
  clearAllHighlights()
  highlightedPhrase.value = phrase
  speak(phrase.en)
}
function handleWordClick(event, word) {
  event.stopPropagation()
  if (highlightedWord.value === word) return
  clearAllHighlights()
  highlightedWord.value = word
  speak(word.w)
  const rect = event.target.getBoundingClientRect()
  let left = rect.right + 8, top = rect.top - 10
  const tooltipWidth = 160
  if (left + tooltipWidth > window.innerWidth - 10) left = rect.left - tooltipWidth - 8
  if (top < 10) top = rect.bottom + 10
  tooltipX.value = left
  tooltipY.value = top
  tooltipWord.value = word
  tooltipVisible.value = true
  if (!wordbook.value.some(w => w.word === word.w)) {
    wordbook.value.push({ word: word.w, meaning: word.m || word.w })
    localStorage.setItem('wordbook', JSON.stringify(wordbook.value))
  }
}
function handleDocumentClick(e) {
  const target = e.target
  if (target.closest('.word') || target.closest('.phrase-item') || target.closest('.module-row') ||
      target.closest('.test-btn') || target.closest('.word-tooltip') || target.closest('.modal')) return
  clearAllHighlights()
}

// 自测
function buildTestItemsFromSentence(sentence) {
  const items = []
  if (!sentence.modules) return items
  for (const mod of sentence.modules) {
    for (const phrase of mod.phrases) {
      for (const word of phrase.words) items.push({ type: 'word', prompt: word.m || word.w, answer: word.w })
      items.push({ type: 'phrase', prompt: phrase.cn, answer: phrase.en })
    }
    const moduleText = mod.phrases.map(p => p.en).join(' ')
    const moduleCn = mod.label
    items.push({ type: 'module', prompt: `【${moduleCn}】请写出完整的模块内容`, answer: moduleText })
  }
  items.push({ type: 'sentence', prompt: sentence.chinese, answer: sentence.english })
  return items
}
function startFullTest() {
  if (!currentSentence.value) return
  testItems.value = buildTestItemsFromSentence(currentSentence.value)
  testItemIndex.value = 0
  consecutiveCorrect = 0
  testMode.value = true
  toastMessage.value = ''
  nextTick(() => testInputRef.value?.focus())
  showCurrentTestItem()
}
function showCurrentTestItem() {
  if (testItemIndex.value >= testItems.value.length) {
    showToast('🎉 恭喜！全句掌握！', 'success')
    playSound(true)
    setTimeout(() => testMode.value = false, 1500)
    return
  }
  const item = testItems.value[testItemIndex.value]
  const typeLabel = item.type === 'word' ? '单词' : (item.type === 'phrase' ? '短语' : (item.type === 'module' ? '模块' : '整句'))
  testPrompt.value = `${typeLabel}：${item.prompt}`
  testInput.value = ''
  speak(item.answer)
  nextTick(() => testInputRef.value?.focus())
}
function submitTest() {
  if (testItemIndex.value >= testItems.value.length) return
  const item = testItems.value[testItemIndex.value]
  const userAns = testInput.value.trim().toLowerCase().replace(/[.,!?]/g, '')
  const correctAns = item.answer.trim().toLowerCase().replace(/[.,!?]/g, '')
  if (userAns === correctAns) {
    playSound(true)
    showToast('✅ 正确！', 'success')
    testItemIndex.value++
    consecutiveCorrect = 0
    setTimeout(() => showCurrentTestItem(), 500)
  } else {
    playSound(false)
    showToast(`❌ 正确答案：${item.answer}`, 'error')
    testInput.value = correctAns
    setTimeout(() => { testInput.value = ''; nextTick(() => testInputRef.value?.focus()) }, 1500)
  }
}
function testPhrase(phrase) {
  if (!collectedPhrases.value.some(p => p.en === phrase.en)) {
    collectedPhrases.value.push({ en: phrase.en, cn: phrase.cn })
    localStorage.setItem('collectedPhrases', JSON.stringify(collectedPhrases.value))
  }
  const items = phrase.words.map(w => ({ type: 'word', prompt: w.m || w.w, answer: w.w }))
  items.push({ type: 'phrase', prompt: phrase.cn, answer: phrase.en })
  testItems.value = items
  testItemIndex.value = 0
  consecutiveCorrect = 0
  testMode.value = true
  toastMessage.value = ''
  nextTick(() => testInputRef.value?.focus())
  showCurrentTestItem()
}
function closeTest() { testMode.value = false; toastMessage.value = '' }

// 句子数据
const currentSentence = computed(() => filteredSentences.value[currentIndex.value] || {})
const currentModules = computed(() => currentSentence.value.modules || [])
async function loadSentences() {
  loading.value = true
  const { data, error } = await supabase.from('sentences').select('*').order('sentence_id', { ascending: true })
  if (error) alert('加载失败：' + error.message)
  else {
    allSentences.value = data || []
    filterLevel(currentLevel.value)
  }
  loading.value = false
}
watch(currentSentence, () => preloadNearbySentences(), { immediate: true })
function editTranslation() { const newCn = prompt('编辑中文翻译', currentSentence.value.chinese); if (newCn) alert('演示模式') }
function cycleTheme() { const themes = ['light', 'dark', 'eyecare']; const idx = themes.indexOf(theme.value); theme.value = themes[(idx + 1) % themes.length] }
function toggleCN() { isCNHidden.value = !isCNHidden.value }
function toggleEN() { isENHidden.value = !isENHidden.value }

// 生词本操作
function openWordbook() { showWordbook.value = true }
function switchTab(tab) { 
  bookTab.value = tab
  selectAllWords.value = false
  selectAllPhrases.value = false
  selectedWordIndices.value = []
  selectedPhraseIndices.value = []
}
function toggleSelectAllWords() {
  selectAllWords.value ? selectedWordIndices.value = wordbook.value.map((_, i) => i) : selectedWordIndices.value = []
}
function toggleSelectAllPhrases() {
  selectAllPhrases.value ? selectedPhraseIndices.value = collectedPhrases.value.map((_, i) => i) : selectedPhraseIndices.value = []
}
function deleteSelectedItems() {
  if (bookTab.value === 'words') {
    const newList = wordbook.value.filter((_, i) => !selectedWordIndices.value.includes(i))
    wordbook.value = newList
    localStorage.setItem('wordbook', JSON.stringify(wordbook.value))
    selectedWordIndices.value = []
    selectAllWords.value = false
  } else {
    const newList = collectedPhrases.value.filter((_, i) => !selectedPhraseIndices.value.includes(i))
    collectedPhrases.value = newList
    localStorage.setItem('collectedPhrases', JSON.stringify(collectedPhrases.value))
    selectedPhraseIndices.value = []
    selectAllPhrases.value = false
  }
  showToast('已删除选中项', 'success')
}
function removeWord(idx) {
  wordbook.value.splice(idx, 1)
  localStorage.setItem('wordbook', JSON.stringify(wordbook.value))
  selectedWordIndices.value = selectedWordIndices.value.filter(i => i !== idx).map(i => i > idx ? i - 1 : i)
}
function removePhrase(idx) {
  collectedPhrases.value.splice(idx, 1)
  localStorage.setItem('collectedPhrases', JSON.stringify(collectedPhrases.value))
  selectedPhraseIndices.value = selectedPhraseIndices.value.filter(i => i !== idx).map(i => i > idx ? i - 1 : i)
}
function startStudyMode(type, idx) {
  if (type === 'word') {
    studyCardList.value = wordbook.value.map(w => ({ word: w.word, meaning: w.meaning }))
    studyCardIndex.value = idx
  } else {
    studyCardList.value = collectedPhrases.value.map(p => ({ word: p.en, meaning: p.cn }))
    studyCardIndex.value = idx
  }
  studyMode.value = true
  nextTick(() => speak(studyCardItem.value.word))
}
function prevStudyCard() { if (studyCardIndex.value > 0) studyCardIndex.value--; speak(studyCardItem.value.word) }
function nextStudyCard() { if (studyCardIndex.value < studyCardList.value.length - 1) studyCardIndex.value++; speak(studyCardItem.value.word) }
function startSpellMode(type, idx) {
  if (type === 'word') {
    spellCardList.value = wordbook.value.map(w => ({ word: w.word, meaning: w.meaning }))
    spellCardIndex.value = idx
  } else {
    spellCardList.value = collectedPhrases.value.map(p => ({ word: p.en, meaning: p.cn }))
    spellCardIndex.value = idx
  }
  spellInput.value = ''
  spellFeedback.value = ''
  spellMode.value = true
  nextTick(() => { speak(spellCardItem.value.word); spellInputRef.value?.focus() })
}
function checkSpell() {
  const current = spellCardItem.value
  const userAns = spellInput.value.trim().toLowerCase().replace(/[.,!?]/g, '')
  const correctAns = current.word.trim().toLowerCase().replace(/[.,!?]/g, '')
  if (userAns === correctAns) {
    playSound(true)
    spellFeedback.value = '✅ 正确！'
    spellFeedbackType.value = 'success'
    setTimeout(() => nextSpellCard(), 1200)
  } else {
    playSound(false)
    spellFeedback.value = `❌ 错误，正确答案：${correctAns}`
    spellFeedbackType.value = 'error'
    spellInput.value = ''
    setTimeout(() => { spellFeedback.value = ''; spellInputRef.value?.focus() }, 1500)
  }
}
function nextSpellCard() {
  if (spellCardIndex.value < spellCardList.value.length - 1) {
    spellCardIndex.value++
    spellInput.value = ''
    spellFeedback.value = ''
    speak(spellCardItem.value.word)
    nextTick(() => spellInputRef.value?.focus())
  } else spellMode.value = false
}
function prevSpellCard() {
  if (spellCardIndex.value > 0) {
    spellCardIndex.value--
    spellInput.value = ''
    spellFeedback.value = ''
    speak(spellCardItem.value.word)
    nextTick(() => spellInputRef.value?.focus())
  }
}
function loadLocalCollections() {
  const saved = localStorage.getItem('wordbook')
  if (saved) wordbook.value = JSON.parse(saved)
  const savedPhrases = localStorage.getItem('collectedPhrases')
  if (savedPhrases) collectedPhrases.value = JSON.parse(savedPhrases)
}

// 预加载语音列表并预热
function initSpeech() {
  warmupSpeech()
  if (window.speechSynthesis.onvoiceschanged !== undefined) {
    window.speechSynthesis.onvoiceschanged = () => {
      getBestVoice()
      warmupSpeech()
    }
  }
}

onMounted(() => {
  loadSentences()
  initSpeech()
  loadLocalCollections()
  document.addEventListener('click', handleDocumentClick)
  window.addEventListener('keydown', handleKeydown)
})
onUnmounted(() => {
  window.removeEventListener('keydown', handleKeydown)
})
</script>

<style>
:root {
  --bg: #f5f7fb;
  --surface: #ffffff;
  --surface2: #f8f9fc;
  --text: #1a1c2a;
  --text2: #4a4d5e;
  --text3: #86899b;
  --border: #e2e4e8;
  --accent: #4f6ef6;
  --sidebar-width: 150px;
}
[data-theme="dark"] {
  --bg: #0f1119;
  --surface: #1a1d28;
  --surface2: #222633;
  --text: #e2e4ec;
  --text2: #b0b5c6;
  --text3: #7c8195;
  --border: #2d3142;
  --accent: #6c8cff;
}
[data-theme="eyecare"] {
  --bg: #fdf6e3;
  --surface: #fefcf3;
  --surface2: #f5ecd7;
  --text: #4a3f35;
  --text2: #6b5b4e;
  --text3: #8b7a68;
  --border: #d5c9b5;
  --accent: #859900;
}
* { margin: 0; padding: 0; box-sizing: border-box; }
body { font-family: system-ui, 'Segoe UI', 'PingFang SC', sans-serif; background: var(--bg); color: var(--text); transition: 0.3s; width: 100%; }
.app { min-height: 100vh; display: flex; flex-direction: column; }

.topbar {
  position: sticky;
  top: 0;
  background: var(--surface);
  border-bottom: 1px solid var(--border);
  padding: 0 16px;
  min-height: 52px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  z-index: 100;
  gap: 12px;
  flex-wrap: nowrap;
  overflow-x: auto;
  overflow-y: hidden;
  scrollbar-width: thin;
}
.topbar-left { display: flex; align-items: center; gap: 10px; flex-shrink: 0; }
.controls { display: flex; align-items: center; gap: 6px; flex-shrink: 0; }
.controls button, .controls select, .page-indicator { flex-shrink: 0; white-space: nowrap; }
.controls button, .controls select {
  padding: 5px 10px;
  border-radius: 20px;
  border: 1px solid var(--border);
  background: var(--surface);
  cursor: pointer;
  font-size: 0.75rem;
  color: var(--text);
}
.page-indicator { font-size: 0.75rem; color: var(--text); }
.menu-toggle { background: none; border: none; font-size: 1.3rem; cursor: pointer; color: var(--text); padding: 0 4px; }
.logo { font-weight: 700; color: var(--accent); font-size: 0.95rem; white-space: nowrap; }
.logo span { background: var(--accent); color: white; padding: 3px 6px; border-radius: 6px; margin-right: 4px; font-size: 0.8rem; }

.main-layout { display: flex; flex: 1; min-height: calc(100vh - 52px); position: relative; }
.main-container { flex: 1; display: flex; flex-direction: column; width: 100%; }

.sidebar {
  position: fixed;
  left: 0;
  top: 52px;
  bottom: 0;
  width: var(--sidebar-width);
  background: var(--surface);
  border-right: 1px solid var(--border);
  transform: translateX(-100%);
  transition: transform 0.2s ease;
  z-index: 90;
  overflow-y: auto;
  padding: 16px 10px;
}
.sidebar-visible { transform: translateX(0); }
.sidebar-section { margin-bottom: 20px; }
.sidebar-title {
  font-size: 0.7rem;
  font-weight: 600;
  color: var(--text);
  margin-bottom: 6px;
  padding: 5px 8px;
  cursor: pointer;
  display: flex;
  justify-content: space-between;
  align-items: center;
  border-radius: 6px;
}
.sidebar-title:hover { background: var(--surface2); }
.toggle-icon { font-size: 0.6rem; color: var(--text3); }
.sidebar-options { padding-left: 10px; }
.sidebar-btn {
  display: flex;
  align-items: center;
  gap: 6px;
  width: 100%;
  padding: 6px 10px;
  border-radius: 6px;
  border: none;
  background: transparent;
  cursor: pointer;
  font-size: 0.8rem;
  color: var(--text2);
  transition: 0.2s;
}
.sidebar-btn:hover { background: var(--surface2); color: var(--text); }

.main-area { flex: 1; padding: 20px; max-width: 1100px; margin: 0 auto; width: 100%; }
.sentence-card { background: var(--surface); border-radius: 16px; padding: 20px; box-shadow: 0 1px 8px rgba(0,0,0,0.05); border: 1px solid var(--border); margin-bottom: 20px; }

.sentence-english {
  font-size: 1.25rem;
  font-weight: 500;
  color: var(--text);
  line-height: 1.6;
  text-align: left;
  word-break: break-word;
  overflow-wrap: break-word;
  white-space: normal;
  text-indent: 2em;
  margin-bottom: 16px;
}

.chinese-translation { background: var(--surface2); padding: 10px 14px; border-radius: 10px; margin: 12px 0; cursor: pointer; color: var(--text); font-size: 0.9rem; font-weight: 500; }
.hidden-cn { color: transparent; position: relative; }
.hidden-cn::after { content: '👆 点击显示中文'; position: absolute; left: 12px; top: 50%; transform: translateY(-50%); color: var(--text3); font-size: 0.75rem; }

.modules-container { margin: 12px 0; }
.module-row { display: flex; flex-wrap: wrap; align-items: baseline; gap: 6px; padding: 10px; margin-bottom: 6px; border-radius: 10px; border-left: 3px solid; cursor: pointer; transition: 0.15s; }
.module-highlight { box-shadow: 0 0 0 2px var(--accent); background-color: rgba(79,110,246,0.2) !important; }
.module-label { font-size: 0.7rem; font-weight: 600; padding: 2px 6px; border-radius: 10px; }
.module-phrases { display: flex; flex-wrap: wrap; align-items: baseline; gap: 8px; flex: 1; }
.phrase-item { display: inline-flex; flex-wrap: wrap; align-items: baseline; gap: 4px; padding: 4px 10px; border-radius: 14px; cursor: pointer; transition: 0.1s; }
.phrase-highlight { box-shadow: 0 0 0 2px var(--accent); background-color: rgba(79,110,246,0.4) !important; }
.word { font-size: 0.9rem; font-weight: 500; cursor: pointer; padding: 1px 2px; border-radius: 4px; color: var(--text); transition: 0.1s; }
.word:hover { background: rgba(0,0,0,0.05); }
.word-highlight { background-color: #ffeb3b !important; box-shadow: 0 0 0 2px #ff9800; border-radius: 4px; }
.hidden-en { color: transparent; background: rgba(0,0,0,0.08); border-radius: 3px; padding: 0 3px; }
.phrase-cn { font-size: 0.75rem; color: var(--text2); margin-left: 3px; font-weight: 400; }
.hidden-cn-text { color: transparent; background: rgba(0,0,0,0.08); border-radius: 3px; padding: 0 3px; }
.phrase-label { font-size: 0.6rem; font-weight: 600; color: var(--text3); background: rgba(0,0,0,0.06); padding: 2px 5px; border-radius: 10px; }
.test-btn { background: none; border: none; cursor: pointer; font-size: 0.7rem; padding: 2px 5px; border-radius: 5px; color: var(--text2); }
.test-btn:hover { background: rgba(0,0,0,0.1); }

.word-tooltip {
  position: fixed;
  background: var(--surface);
  border: 1px solid var(--accent);
  border-radius: 10px;
  padding: 6px 10px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.15);
  z-index: 1000;
  min-width: 100px;
  max-width: 200px;
  text-align: left;
  pointer-events: none;
  word-break: break-word;
}
.tooltip-word { font-weight: 700; font-size: 0.9rem; color: var(--accent); }
.tooltip-pos { font-size: 0.65rem; color: var(--text3); margin-top: 2px; }
.tooltip-meaning { font-size: 0.75rem; color: var(--text); margin-top: 3px; }

.test-modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0,0,0,0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 300;
}
.test-modal {
  background: var(--surface);
  border-radius: 24px;
  width: 90%;
  max-width: 500px;
  box-shadow: 0 20px 40px rgba(0,0,0,0.25);
}
.test-modal-header { display: flex; justify-content: space-between; align-items: center; padding: 16px 20px; border-bottom: 1px solid var(--border); }
.test-modal-header h3 { margin: 0; color: var(--accent); font-size: 1.2rem; }
.test-modal-close { background: none; border: none; font-size: 1.3rem; cursor: pointer; color: var(--text2); }
.test-modal-body { padding: 20px; text-align: center; }
.test-prompt-card { background: var(--surface2); border-radius: 16px; padding: 14px 18px; margin-bottom: 16px; display: flex; align-items: center; justify-content: center; gap: 10px; border: 1px solid var(--border); }
.test-prompt-icon { font-size: 1.6rem; }
.test-prompt-text { font-size: 1rem; font-weight: 600; color: var(--text); line-height: 1.4; }
.toast-message { margin: 6px 0; padding: 6px 10px; border-radius: 30px; font-size: 0.8rem; text-align: center; }
.toast-message.success { background: #e6f7e6; color: #2e7d32; border: 1px solid #a5d6a7; }
.toast-message.error { background: #ffebee; color: #c62828; border: 1px solid #ef9a9a; }
.test-input { width: 100%; padding: 10px 14px; font-size: 0.9rem; border: 2px solid var(--border); border-radius: 16px; background: var(--surface); color: var(--text); outline: none; text-align: center; }
.test-input:focus { border-color: var(--accent); }
.test-buttons { display: flex; gap: 12px; justify-content: center; margin-top: 20px; }
.test-submit, .test-cancel { padding: 8px 24px; border-radius: 30px; border: none; cursor: pointer; font-size: 0.85rem; font-weight: 600; }
.test-submit { background: var(--accent); color: white; }
.test-cancel { background: var(--surface2); color: var(--text2); border: 1px solid var(--border); }

.loading { text-align: center; padding: 40px; color: var(--text2); }

.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0,0,0,0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 200;
}
.modal {
  background: var(--surface);
  border-radius: 16px;
  padding: 20px;
  width: 90%;
  max-width: 500px;
  max-height: 70vh;
  overflow-y: auto;
}
.modal-large { max-width: 550px; }
.modal h3 { margin-bottom: 14px; }
.book-tabs { display: flex; gap: 8px; margin-bottom: 14px; }
.book-tabs button { padding: 5px 14px; border-radius: 20px; border: 1px solid var(--border); background: var(--surface); cursor: pointer; font-size: 0.8rem; }
.book-tabs button.active { background: var(--accent); color: white; border-color: var(--accent); }
.book-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 10px; gap: 8px; flex-wrap: wrap; }
.select-all-label { display: flex; align-items: center; gap: 5px; font-size: 0.8rem; cursor: pointer; }
.delete-selected { padding: 3px 10px; border-radius: 14px; border: 1px solid #d63e4a; background: #fff0f0; cursor: pointer; font-size: 0.7rem; color: #d63e4a; }
.delete-selected:disabled { opacity: 0.4; cursor: not-allowed; }
.book-list { max-height: 250px; overflow-y: auto; }
.book-item { display: flex; align-items: center; gap: 8px; padding: 6px 0; border-bottom: 1px solid var(--border); flex-wrap: wrap; }
.book-checkbox { width: 16px; height: 16px; cursor: pointer; }
.book-num { font-size: 0.75rem; color: var(--text3); min-width: 30px; }
.book-content { flex: 1; font-size: 0.8rem; color: var(--text); }
.book-study, .book-spell, .book-delete { background: none; border: none; cursor: pointer; font-size: 0.9rem; margin: 0 2px; }
.book-delete { color: #d63e4a; }
.empty-tip { text-align: center; padding: 20px; color: var(--text3); font-size: 0.85rem; }
.close-modal { margin-top: 14px; padding: 6px 16px; border-radius: 20px; border: 1px solid var(--border); background: var(--surface); cursor: pointer; width: 100%; }

.study-modal, .spell-modal {
  background: var(--surface);
  border-radius: 24px;
  width: 90%;
  max-width: 450px;
  padding: 20px;
  text-align: center;
}
.study-modal-header, .spell-modal-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 14px; }
.study-card, .spell-card { background: var(--surface2); border-radius: 16px; padding: 24px 16px; margin: 16px 0; }
.study-card-word { font-size: 1.4rem; font-weight: 700; color: var(--accent); margin-bottom: 10px; }
.study-card-meaning, .spell-card-hint { font-size: 1rem; color: var(--text); margin-bottom: 12px; }
.study-card-speak, .spell-check, .spell-skip { margin-top: 8px; padding: 6px 16px; border-radius: 25px; border: none; background: var(--accent); color: white; cursor: pointer; font-size: 0.8rem; }
.study-nav, .spell-nav { display: flex; justify-content: center; gap: 16px; margin: 12px 0; }
.spell-input { width: 100%; padding: 10px; font-size: 0.9rem; border: 2px solid var(--border); border-radius: 14px; margin-top: 12px; text-align: center; background: var(--surface); color: var(--text); }
.spell-feedback { margin-top: 10px; font-size: 0.8rem; }
.spell-feedback.success { color: #2e7d32; }
.spell-feedback.error { color: #c62828; }
.spell-buttons { display: flex; gap: 10px; justify-content: center; margin-top: 10px; }
.study-exit, .spell-exit { margin-top: 16px; padding: 6px 16px; border-radius: 25px; border: 1px solid var(--border); background: var(--surface); cursor: pointer; }

@media screen and (max-width: 768px) {
  .topbar { padding: 0 10px; gap: 8px; }
  .controls button, .controls select { padding: 3px 6px; font-size: 0.65rem; }
  .page-indicator { font-size: 0.65rem; }
  .logo { font-size: 0.85rem; }
  .main-area { padding: 12px; }
  .sentence-card { padding: 14px; }
  .sentence-english { font-size: 1.1rem; text-indent: 1.5em; }
  .module-row { padding: 8px; }
  .phrase-item { padding: 3px 8px; }
  .word { font-size: 0.85rem; }
}
</style>
