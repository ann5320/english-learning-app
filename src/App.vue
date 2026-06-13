<template>
  <div class="app" :data-theme="theme">
    <header class="topbar">
      <div class="topbar-left">
        <button class="menu-toggle" @click="toggleSidebar">☰</button>
        <div class="logo logo-purple" @click="goHome">
          <span class="logo-bg">ME 模块英语</span>
        </div>
      </div>
      <div class="controls">
        <button @click="prev">◀</button>
        <span class="page-indicator">{{ currentIndex + 1 }} / {{ filteredSentences.length }}</span>
        <button @click="next">▶</button>
        <button @click="openWordbook">📖 收藏</button>
        <button @click="startFullTest" :disabled="isTestMode">✏️ 拼写</button>
        <button class="settings-btn" @click="openSettings">⚙️ 设置</button>
      </div>
    </header>

    <!-- 设置面板 -->
    <Teleport to="body">
      <div v-if="settingsVisible" class="settings-overlay" @click="closeSettings">
        <div class="settings-panel" @click.stop>
          <div class="settings-header">
            <h3>设置</h3>
            <button class="settings-close" @click="closeSettings">✕</button>
          </div>
          <div class="settings-content">
            <div class="setting-group">
              <div class="setting-title">📖 显示设置</div>
              <label class="setting-item">
                <input type="checkbox" v-model="isCNHidden" @change="saveSettings">
                <span>隐藏中文</span>
              </label>
              <label class="setting-item">
                <input type="checkbox" v-model="isENHidden" @change="saveSettings">
                <span>隐藏英文</span>
              </label>
            </div>

            <div class="setting-group">
              <div class="setting-title">🔊 语音设置</div>
              <div class="setting-row">
                <span class="setting-label">语速：</span>
                <div class="speed-buttons">
                  <button 
                    v-for="s in speedOptions" 
                    :key="s"
                    class="speed-btn"
                    :class="{ active: speed === s }"
                    @click="setSpeed(s)"
                  >{{ s }}x</button>
                </div>
              </div>
              <div class="setting-row">
                <span class="setting-label">发音：</span>
                <div class="voice-buttons">
                  <button 
                    class="voice-btn"
                    :class="{ active: voiceType === 'en-UK' }"
                    @click="setVoiceType('en-UK')"
                  >🇬🇧 英音</button>
                  <button 
                    class="voice-btn"
                    :class="{ active: voiceType === 'en-US' }"
                    @click="setVoiceType('en-US')"
                  >🇺🇸 美音</button>
                </div>
              </div>
            </div>

            <div class="setting-group">
              <div class="setting-title">🎨 主题</div>
              <div class="theme-buttons">
                <button 
                  class="theme-btn"
                  :class="{ active: theme === 'light' }"
                  @click="setTheme('light')"
                >☀️ 浅色</button>
                <button 
                  class="theme-btn"
                  :class="{ active: theme === 'dark' }"
                  @click="setTheme('dark')"
                >🌙 深色</button>
                <button 
                  class="theme-btn"
                  :class="{ active: theme === 'eyecare' }"
                  @click="setTheme('eyecare')"
                >🌿 护眼</button>
              </div>
            </div>
          </div>
        </div>
      </div>
    </Teleport>

    <div class="main-layout" @click="closeSidebarOnClickOutside">
      <aside class="sidebar" :class="{ 'sidebar-visible': sidebarVisible }">
        <div class="sidebar-inner">
          <div class="sidebar-section">
            <div class="sidebar-title">📚 句库</div>
            <div class="sidebar-options">
              <button class="sidebar-btn" @click="filterLevel('collected')">⭐ 收藏</button>
              <button class="sidebar-btn" @click="filterLevel('beginner')">🟢 初级</button>
              <button class="sidebar-btn" @click="filterLevel('intermediate')">🟡 中级</button>
              <button class="sidebar-btn" @click="filterLevel('advanced')">🔴 四级</button>
            </div>
          </div>

          <div class="sidebar-section">
            <div class="sidebar-title">📖 语法</div>
            <div class="sidebar-options">
              <button class="sidebar-btn" @click="showSyntax = true">句法</button>
              <button class="sidebar-btn" @click="showGrammar = true">词法</button>
            </div>
          </div>

          <div class="sidebar-section">
            <div class="sidebar-title">🔧 拆解器</div>
            <div class="sidebar-options">
              <div class="empty-tip-small">功能开发中...</div>
            </div>
          </div>
        </div>
      </aside>

      <div class="main-container">
        <main class="main-area" ref="mainAreaRef" @touchstart="onTouchStart" @touchend="onTouchEnd">
          <div v-if="loading" class="loading">加载句子中...</div>
          <div v-else-if="filteredSentences.length === 0" class="loading">暂无句子，请检查数据库</div>
          <div v-else class="sentence-card">
            <!-- 红色心形收藏按钮 -->
            <button class="star-collect-btn" @click.stop="toggleCurrentSentenceCollect">
              <span class="star-icon" :class="{ 'star-filled': isCurrentSentenceCollected }">
                {{ isCurrentSentenceCollected ? '❤️' : '♡' }}
              </span>
            </button>
            <div class="sentence-english" @click="speakFull">
              {{ currentSentence.english }}
            </div>
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

    <!-- 拼写弹窗 -->
    <div v-if="testMode" class="test-modal-overlay" @click.self="closeTest">
      <div class="test-modal">
        <div class="test-modal-header">
          <h3>✏️ 拼写练习</h3>
          <div class="test-nav">
            <button @click="prevTestSentence" :disabled="testSentenceIndex === 0" class="test-nav-btn">◀</button>
            <span class="test-sentence-index">{{ testSentenceIndex + 1 }} / {{ filteredSentences.length }}</span>
            <button @click="nextTestSentence" :disabled="testSentenceIndex === filteredSentences.length - 1" class="test-nav-btn">▶</button>
          </div>
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

    <!-- 收藏弹窗 -->
    <div v-if="showWordbook" class="modal-overlay" @click.self="showWordbook = false">
      <div class="modal modal-large">
        <h3>📖 收藏</h3>
        <div class="book-tabs">
          <button :class="{ active: bookTab === 'words' }" @click="switchTab('words')">单词 ({{ wordbook.length }})</button>
          <button :class="{ active: bookTab === 'phrases' }" @click="switchTab('phrases')">短语 ({{ collectedPhrases.length }})</button>
          <button :class="{ active: bookTab === 'sentences' }" @click="switchTab('sentences')">句子 ({{ collectedSentences.length }})</button>
        </div>
        <div class="book-header">
          <label class="select-all-label">
            <input type="checkbox" v-model="selectAll" @change="toggleSelectAll" />
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
          <div v-if="bookTab === 'sentences'">
            <div v-if="collectedSentences.length === 0" class="empty-tip">暂无收藏句子</div>
            <div v-for="(item, idx) in collectedSentences" :key="idx" class="book-item">
              <input type="checkbox" :value="idx" v-model="selectedSentenceIndices" class="book-checkbox" />
              <span class="book-num">{{ idx + 1 }}.</span>
              <span class="book-content sentence-content-preview">{{ item.english?.substring(0, 50) }}{{ item.english?.length > 50 ? '...' : '' }}</span>
              <button class="book-study" @click="startStudyMode('sentence', idx)">📖</button>
              <button class="book-spell" @click="startSpellMode('sentence', idx)">✏️</button>
              <button class="book-delete" @click="removeSentence(idx)">🗑</button>
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
          <div class="study-card-word">{{ studyCardItem.word || studyCardItem.en || studyCardItem.english }}</div>
          <div class="study-card-meaning">{{ studyCardItem.meaning || studyCardItem.cn || studyCardItem.chinese }}</div>
          <button class="study-card-speak" @click="speak(studyCardItem.word || studyCardItem.en || studyCardItem.english)">🔊</button>
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
          <div class="spell-card-hint">{{ spellCardItem.meaning || spellCardItem.cn || spellCardItem.chinese }}</div>
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
    <div v-if="showSyntax" class="modal-overlay" @click.self="showSyntax = false">
      <div class="modal"><h3>📖 句法</h3><p>功能开发中...</p><button @click="showSyntax = false">关闭</button></div>
    </div>
    <div v-if="showGrammar" class="modal-overlay" @click.self="showGrammar = false">
      <div class="modal"><h3>📚 词法</h3><p>功能开发中...</p><button @click="showGrammar = false">关闭</button></div>
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

// 设置相关
const settingsVisible = ref(false)
const theme = ref('light')
const isCNHidden = ref(false)
const isENHidden = ref(false)
const speed = ref(1)
const voiceType = ref('en-US')
const speedOptions = [0.6, 1, 1.3, 1.5]

// 侧边栏
const currentLevel = ref('collected')
const sidebarVisible = ref(false)

const showSyntax = ref(false)
const showGrammar = ref(false)

// 拼写模式
const testMode = ref(false)
const testSentenceIndex = ref(0)
const testCurrentSentence = ref(null)
const testItems = ref([])
const testItemIndex = ref(0)
const testInput = ref('')
const testPrompt = ref('')
const testInputRef = ref(null)
const toastMessage = ref('')
const toastType = ref('')
let toastTimerId = null
const passedItems = ref(new Set())
const isTestMode = ref(false)

// 收藏
const showWordbook = ref(false)
const bookTab = ref('words')
const wordbook = ref([])
const collectedPhrases = ref([])
const collectedSentences = ref([])
const selectedWordIndices = ref([])
const selectedPhraseIndices = ref([])
const selectedSentenceIndices = ref([])
const selectAll = ref(false)

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
  if (bookTab.value === 'phrases') return selectedPhraseIndices.value.length
  return selectedSentenceIndices.value.length
})

const isCurrentSentenceCollected = computed(() => {
  if (!currentSentence.value?.id) return false
  return collectedSentences.value.some(s => s.id === currentSentence.value.id)
})

function toggleCurrentSentenceCollect() {
  if (!currentSentence.value?.id) return
  if (isCurrentSentenceCollected.value) {
    const index = collectedSentences.value.findIndex(s => s.id === currentSentence.value.id)
    if (index !== -1) removeSentence(index)
  } else {
    addSentence(currentSentence.value)
  }
}

function addSentence(sentence) {
  if (!collectedSentences.value.some(s => s.id === sentence.id)) {
    collectedSentences.value.unshift({
      id: sentence.id,
      english: sentence.english,
      chinese: sentence.chinese,
      level: sentence.level,
      collectedAt: Date.now()
    })
    saveSentencesToLocal()
  }
}

function removeSentence(idx) {
  collectedSentences.value.splice(idx, 1)
  saveSentencesToLocal()
}

function saveSentencesToLocal() {
  localStorage.setItem('collectedSentences', JSON.stringify(collectedSentences.value))
}

function loadCollectedSentences() {
  const saved = localStorage.getItem('collectedSentences')
  if (saved) {
    collectedSentences.value = JSON.parse(saved)
  }
}

// 语音
let cachedUtterance = null
let cachedVoice = null
let speechWarmedUp = false

function getBestVoice() {
  if (cachedVoice) return cachedVoice
  const voices = window.speechSynthesis.getVoices()
  const preferredNames = ['Microsoft', 'Samantha', 'David', 'Zira', 'Alex', 'Siri']
  let bestVoice = voices.find(v => 
    v.lang === voiceType.value && 
    preferredNames.some(name => v.name.includes(name))
  )
  if (!bestVoice) bestVoice = voices.find(v => v.lang === voiceType.value)
  if (!bestVoice) bestVoice = voices.find(v => v.lang === 'en-US')
  cachedVoice = bestVoice
  return cachedVoice
}

function warmupSpeech() {
  if (speechWarmedUp) return
  try {
    window.speechSynthesis.getVoices()
    getBestVoice()
    const utterance = new SpeechSynthesisUtterance('')
    utterance.volume = 0
    window.speechSynthesis.speak(utterance)
    speechWarmedUp = true
  } catch(e) { console.warn('语音预热失败', e) }
}

function speak(text) {
  if (!text) return
  if (window.speechSynthesis.speaking) window.speechSynthesis.cancel()
  if (!cachedUtterance) {
    cachedUtterance = new SpeechSynthesisUtterance()
    cachedUtterance.lang = voiceType.value
  }
  cachedUtterance.text = text
  cachedUtterance.rate = speed.value
  cachedUtterance.voice = getBestVoice()
  window.speechSynthesis.speak(cachedUtterance)
}

function speakFull() { 
  if (currentSentence.value.english) {
    speak(currentSentence.value.english)
  }
}

function saveSettings() {
  localStorage.setItem('theme', theme.value)
  localStorage.setItem('hideChinese', isCNHidden.value)
  localStorage.setItem('hideEnglish', isENHidden.value)
  localStorage.setItem('speed', speed.value)
  localStorage.setItem('voiceType', voiceType.value)
}

function loadSettings() {
  const savedTheme = localStorage.getItem('theme')
  if (savedTheme) theme.value = savedTheme
  const savedHideChinese = localStorage.getItem('hideChinese')
  if (savedHideChinese !== null) isCNHidden.value = savedHideChinese === 'true'
  const savedHideEnglish = localStorage.getItem('hideEnglish')
  if (savedHideEnglish !== null) isENHidden.value = savedHideEnglish === 'true'
  const savedSpeed = localStorage.getItem('speed')
  if (savedSpeed) speed.value = parseFloat(savedSpeed)
  const savedVoice = localStorage.getItem('voiceType')
  if (savedVoice) voiceType.value = savedVoice
  document.documentElement.setAttribute('data-theme', theme.value)
}

function setTheme(val) {
  theme.value = val
  document.documentElement.setAttribute('data-theme', val)
  saveSettings()
}

function setSpeed(val) {
  speed.value = val
  saveSettings()
}

function setVoiceType(val) {
  voiceType.value = val
  cachedVoice = null
  getBestVoice()
  saveSettings()
}

function openSettings() { settingsVisible.value = true }
function closeSettings() { settingsVisible.value = false }

function goHome() {}

// 辅助
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
  if (level === 'collected') {
    const collectedIds = collectedSentences.value.map(s => s.id)
    filteredSentences.value = allSentences.value.filter(s => collectedIds.includes(s.id))
  } else if (level === 'all') {
    filteredSentences.value = [...allSentences.value]
  } else {
    filteredSentences.value = allSentences.value.filter(s => s.level === level)
  }
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
function handleKeydown(e) { if (e.key === 'ArrowLeft') prev(); if (e.key === 'ArrowRight') next() }
function onTouchStart(e) { touchStartX.value = e.touches[0].clientX }
function onTouchEnd(e) {
  const deltaX = e.changedTouches[0].clientX - touchStartX.value
  if (Math.abs(deltaX) > 50) deltaX > 0 ? prev() : next()
}

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
      target.closest('.test-btn') || target.closest('.word-tooltip') || target.closest('.modal') ||
      target.closest('.star-collect-btn')) return
  clearAllHighlights()
}

// 拼写顺序
function buildTestItemsFromSentence(sentence) {
  const items = []
  const seenContent = new Set()
  
  if (!sentence.modules) return items
  
  for (const mod of sentence.modules) {
    for (const phrase of mod.phrases) {
      for (const word of phrase.words) {
        const content = word.w.toLowerCase()
        if (!seenContent.has(content)) {
          seenContent.add(content)
          items.push({ 
            type: 'word', 
            prompt: word.m || word.w, 
            answer: word.w, 
            key: `word:${content}`
          })
        }
      }
      const phraseContent = phrase.en.toLowerCase()
      if (!seenContent.has(phraseContent)) {
        seenContent.add(phraseContent)
        items.push({ 
          type: 'phrase', 
          prompt: phrase.cn, 
          answer: phrase.en, 
          key: `phrase:${phraseContent}`
        })
      }
    }
  }
  
  const sentenceKey = sentence.english.toLowerCase()
  if (!seenContent.has(sentenceKey)) {
    items.push({ 
      type: 'sentence', 
      prompt: sentence.chinese, 
      answer: sentence.english, 
      key: `sentence:${sentenceKey}`
    })
  }
  
  return items
}

function startFullTest() {
  if (!currentSentence.value) return
  isTestMode.value = true
  testSentenceIndex.value = currentIndex.value
  loadTestForSentence(currentSentence.value)
}

function loadTestForSentence(sentence) {
  testCurrentSentence.value = sentence
  passedItems.value.clear()
  testItems.value = buildTestItemsFromSentence(sentence)
  testItemIndex.value = 0
  testInput.value = ''
  toastMessage.value = ''
  testMode.value = true
  nextTick(() => {
    testInputRef.value?.focus()
    showCurrentTestItem()
  })
}

function prevTestSentence() {
  if (testSentenceIndex.value > 0) {
    testSentenceIndex.value--
    loadTestForSentence(filteredSentences.value[testSentenceIndex.value])
  }
}

function nextTestSentence() {
  if (testSentenceIndex.value < filteredSentences.value.length - 1) {
    testSentenceIndex.value++
    loadTestForSentence(filteredSentences.value[testSentenceIndex.value])
  }
}

function showCurrentTestItem() {
  while (testItemIndex.value < testItems.value.length && 
         passedItems.value.has(testItems.value[testItemIndex.value].key)) {
    testItemIndex.value++
  }
  
  if (testItemIndex.value >= testItems.value.length) {
    showToast('🎉 恭喜！本句掌握！', 'success')
    playSound(true)
    setTimeout(() => {
      if (testSentenceIndex.value < filteredSentences.value.length - 1) {
        nextTestSentence()
      } else {
        testMode.value = false
        isTestMode.value = false
      }
    }, 1500)
    return
  }
  
  const item = testItems.value[testItemIndex.value]
  const typeLabel = item.type === 'word' ? '单词' : (item.type === 'phrase' ? '短语' : '整句')
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
    passedItems.value.add(item.key)
    testItemIndex.value++
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
  
  const items = []
  const seen = new Set()
  
  for (const word of phrase.words) {
    const wordKey = word.w.toLowerCase()
    if (!seen.has(wordKey)) {
      seen.add(wordKey)
      items.push({ type: 'word', prompt: word.m || word.w, answer: word.w, key: wordKey })
    }
  }
  
  const phraseKey = phrase.en.toLowerCase()
  if (!seen.has(phraseKey)) {
    seen.add(phraseKey)
    items.push({ type: 'phrase', prompt: phrase.cn, answer: phrase.en, key: phraseKey })
  }
  
  passedItems.value.clear()
  testItems.value = items
  testItemIndex.value = 0
  testMode.value = true
  isTestMode.value = true
  toastMessage.value = ''
  nextTick(() => testInputRef.value?.focus())
  showCurrentTestItem()
}

function closeTest() { 
  testMode.value = false
  isTestMode.value = false
  toastMessage.value = ''
}

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

function openWordbook() { showWordbook.value = true }
function switchTab(tab) { 
  bookTab.value = tab
  selectAll.value = false
  selectedWordIndices.value = []
  selectedPhraseIndices.value = []
  selectedSentenceIndices.value = []
}
function toggleSelectAll() {
  selectAll.value = !selectAll.value
  if (bookTab.value === 'words') {
    selectedWordIndices.value = selectAll.value ? wordbook.value.map((_, i) => i) : []
  } else if (bookTab.value === 'phrases') {
    selectedPhraseIndices.value = selectAll.value ? collectedPhrases.value.map((_, i) => i) : []
  } else {
    selectedSentenceIndices.value = selectAll.value ? collectedSentences.value.map((_, i) => i) : []
  }
}
function deleteSelectedItems() {
  if (bookTab.value === 'words') {
    const newList = wordbook.value.filter((_, i) => !selectedWordIndices.value.includes(i))
    wordbook.value = newList
    localStorage.setItem('wordbook', JSON.stringify(wordbook.value))
    selectedWordIndices.value = []
  } else if (bookTab.value === 'phrases') {
    const newList = collectedPhrases.value.filter((_, i) => !selectedPhraseIndices.value.includes(i))
    collectedPhrases.value = newList
    localStorage.setItem('collectedPhrases', JSON.stringify(collectedPhrases.value))
    selectedPhraseIndices.value = []
  } else {
    const newList = collectedSentences.value.filter((_, i) => !selectedSentenceIndices.value.includes(i))
    collectedSentences.value = newList
    saveSentencesToLocal()
    selectedSentenceIndices.value = []
  }
  selectAll.value = false
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
  } else if (type === 'phrase') {
    studyCardList.value = collectedPhrases.value.map(p => ({ word: p.en, meaning: p.cn }))
    studyCardIndex.value = idx
  } else {
    studyCardList.value = collectedSentences.value.map(s => ({ word: s.english, meaning: s.chinese }))
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
  } else if (type === 'phrase') {
    spellCardList.value = collectedPhrases.value.map(p => ({ word: p.en, meaning: p.cn }))
    spellCardIndex.value = idx
  } else {
    spellCardList.value = collectedSentences.value.map(s => ({ word: s.english, meaning: s.chinese }))
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
  loadCollectedSentences()
}

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
  loadSettings()
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
  --sidebar-width: 100px;
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

/* Logo */
.logo-purple { cursor: pointer; transition: opacity 0.2s; }
.logo-purple:hover { opacity: 0.85; }
.logo-bg {
  background: #5052f0 !important;
  color: white;
  padding: 2px 7px;
  border-radius: 8px;
  font-size: 0.7rem;
  font-weight: 900;
  letter-spacing: 0.3px;
  display: inline-block;
  box-shadow: 0 1px 3px rgba(99, 102, 241, 0.25);
}

/* 红色心形收藏按钮 */
.star-collect-btn {
  position: absolute;
  top: 18px;
  right: 18px;
  background: none;
  border: none;
  cursor: pointer;
  z-index: 10;
  padding: 6px;
  border-radius: 50%;
  transition: all 0.2s ease;
  display: flex;
  align-items: center;
  justify-content: center;
}
.star-collect-btn:hover {
  background: rgba(0, 0, 0, 0.06);
  transform: scale(1.05);
}
.star-icon {
  font-size: 20px;
  font-weight: normal;
  color: #94a3b8;
  transition: all 0.2s ease;
}
.star-icon.star-filled {
  color: #ec0a0a;
}
.sentence-card { position: relative; }

/* 侧边栏 */
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
  padding: 16px 8px;
}
.sidebar-visible { transform: translateX(0); }
.sidebar-section { margin-bottom: 20px; }
.sidebar-title {
  font-size: 0.7rem;
  font-weight: 600;
  color: var(--text);
  margin-bottom: 8px;
  padding: 5px 8px;
  border-radius: 6px;
  background: var(--surface2);
}
.sidebar-options { padding-left: 0; }
.sidebar-btn {
  display: flex;
  align-items: center;
  gap: 6px;
  width: 100%;
  padding: 6px 8px;
  border-radius: 6px;
  border: none;
  background: transparent;
  cursor: pointer;
  font-size: 0.7rem;
  color: var(--text2);
  transition: 0.2s;
}
.sidebar-btn:hover { background: var(--surface2); color: var(--text); }

/* 设置面板 */
.settings-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0,0,0,0.5);
  z-index: 1000;
  display: flex;
  justify-content: flex-end;
}
.settings-panel {
  width: 280px;
  background: var(--surface);
  height: 100%;
  display: flex;
  flex-direction: column;
  animation: slideIn 0.3s ease;
  box-shadow: -2px 0 10px rgba(0,0,0,0.1);
}
@keyframes slideIn {
  from { transform: translateX(100%); }
  to { transform: translateX(0); }
}
.settings-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 16px 20px;
  border-bottom: 1px solid var(--border);
}
.settings-header h3 { margin: 0; color: var(--accent); }
.settings-close { background: none; border: none; font-size: 20px; cursor: pointer; color: var(--text2); }
.settings-content {
  flex: 1;
  padding: 20px;
  overflow-y: auto;
}
.setting-group { margin-bottom: 24px; }
.setting-title {
  font-size: 14px;
  font-weight: 600;
  color: var(--text);
  margin-bottom: 12px;
  padding-bottom: 6px;
  border-bottom: 1px solid var(--border);
}
.setting-item {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-bottom: 10px;
  cursor: pointer;
  font-size: 14px;
  color: var(--text);
}
.setting-item input { width: 18px; height: 18px; cursor: pointer; }
.setting-row {
  display: flex;
  align-items: center;
  gap: 12px;
  margin-bottom: 12px;
  flex-wrap: wrap;
}
.setting-label { font-size: 13px; color: var(--text2); min-width: 40px; }
.speed-buttons, .voice-buttons, .theme-buttons { display: flex; gap: 8px; flex-wrap: wrap; }
.speed-btn, .voice-btn, .theme-btn {
  padding: 6px 12px;
  border-radius: 20px;
  border: 1px solid var(--border);
  background: var(--surface);
  cursor: pointer;
  font-size: 13px;
  transition: all 0.2s;
}
.speed-btn.active, .voice-btn.active, .theme-btn.active {
  background: var(--accent);
  color: white;
  border-color: var(--accent);
}
.empty-tip-small { padding: 12px; text-align: center; font-size: 12px; color: var(--text3); }
.sentence-content-preview {
  font-size: 0.75rem;
  line-height: 1.4;
  max-width: 300px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
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
.settings-btn {
  background: var(--accent);
  color: white;
  border-color: var(--accent);
}
.page-indicator { font-size: 0.75rem; color: var(--text); }
.menu-toggle { background: none; border: none; font-size: 1.3rem; cursor: pointer; color: var(--text); padding: 0 4px; }
.logo { font-weight: 700; font-size: 0.95rem; white-space: nowrap; display: flex; align-items: center; gap: 2px; }

.main-layout { display: flex; flex: 1; min-height: calc(100vh - 52px); position: relative; }
.main-container { flex: 1; display: flex; flex-direction: column; width: 100%; }
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
  cursor: pointer;
  transition: opacity 0.2s;
  padding-right: 50px;
}
.sentence-english:hover { opacity: 0.8; }

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
.test-modal-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 16px 20px;
  border-bottom: 1px solid var(--border);
}
.test-modal-header h3 { margin: 0; color: var(--accent); font-size: 1.2rem; }
.test-nav {
  display: flex;
  align-items: center;
  gap: 12px;
}
.test-nav-btn {
  background: var(--surface2);
  border: 1px solid var(--border);
  border-radius: 20px;
  padding: 4px 10px;
  cursor: pointer;
  font-size: 14px;
}
.test-nav-btn:disabled { opacity: 0.5; cursor: not-allowed; }
.test-sentence-index { font-size: 13px; color: var(--text2); min-width: 50px; text-align: center; }
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
  .logo-bg { padding: 2px 6px; font-size: 0.6rem; }
  .main-area { padding: 12px; }
  .sentence-card { padding: 14px; }
  .sentence-english { font-size: 1.1rem; text-indent: 1.5em; }
  .module-row { padding: 8px; }
  .phrase-item { padding: 3px 8px; }
  .word { font-size: 0.85rem; }
  .settings-panel { width: 260px; }
  .star-collect-btn { top: 10px; right: 10px; }
  .star-icon { font-size: 18px; }
  .sidebar { width: 80px; }
  .sidebar-btn { font-size: 0.6rem; padding: 4px 6px; }
  .sidebar-title { font-size: 0.6rem; }
}
</style>