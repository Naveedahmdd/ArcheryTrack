<script setup>
import { computed, nextTick, onMounted, ref, watch } from 'vue'

const STORE_KEY = 'arrowtrack-v1'
const initial = {
  players: [{ id: 1, name: 'Alex Morgan' }, { id: 2, name: 'Jamie Lee' }],
  history: [],
  nextPlayerId: 3,
  nextGameId: 1
}
function loadStore() {
  try { return { ...initial, ...JSON.parse(localStorage.getItem(STORE_KEY) || '{}') } }
  catch { return structuredClone(initial) }
}
const store = ref(loadStore())
watch(store, value => localStorage.setItem(STORE_KEY, JSON.stringify(value)), { deep: true })

const page = ref('home')
const mode = ref('solo')
const roundType = ref('challenge')
const distance = ref(30)
const arrowLimit = computed(() => roundType.value === 'practice' ? 12 : 6)
const selectedPlayerId = ref(store.value.players[0]?.id ?? null)
const playerName = ref('')
const newPlayerName = ref('')
const selectedPlayers = ref([])
const currentTurn = ref(0)
const gamePlayers = ref([])
const activePlayer = computed(() => gamePlayers.value[currentTurn.value] || null)
const shots = ref([])
const gameOver = ref(false)
const message = ref('Drag from the bow towards the target, then release to shoot.')
const canvas = ref(null)
const canvasWrap = ref(null)
const canvasWidth = ref(900)
const canvasHeight = ref(430)
const dragging = ref(false)
const dragPoint = ref(null)
const aimPoint = ref(null)
const lastShot = ref(null)
const soundOn = ref(false)
const showAddPlayer = ref(false)
const selectedTab = ref('all')

const totalScore = computed(() => shots.value.reduce((sum, shot) => sum + shot.score, 0))
const arrowsLeft = computed(() => Math.max(0, arrowLimit.value - shots.value.length))
const averageScore = computed(() => shots.value.length ? (totalScore.value / shots.value.length).toFixed(1) : '0.0')
const sortedPlayers = computed(() => [...store.value.players].sort((a,b) => bestScore(b.id) - bestScore(a.id)))
const filteredHistory = computed(() => {
  const rows = [...store.value.history].sort((a,b) => new Date(b.date) - new Date(a.date))
  return selectedTab.value === 'all' ? rows : rows.filter(row => row.mode === selectedTab.value)
})
const personalBest = id => Math.max(0, ...store.value.history.filter(x => x.playerId === id).map(x => x.score))
function bestScore(id) { return personalBest(id) }
function playerById(id) { return store.value.players.find(p => p.id === id)?.name || 'Unknown player' }

function addPlayer() {
  const name = newPlayerName.value.trim()
  if (!name) return
  const exists = store.value.players.some(p => p.name.toLowerCase() === name.toLowerCase())
  if (exists) { message.value = 'That player name already exists.'; return }
  const player = { id: store.value.nextPlayerId++, name }
  store.value.players.push(player)
  selectedPlayerId.value = player.id
  newPlayerName.value = ''
  showAddPlayer.value = false
}
function removePlayer(id) {
  if (store.value.players.length <= 1) { message.value = 'Keep at least one player profile.'; return }
  if (store.value.history.some(h => h.playerId === id)) { message.value = 'This player has saved games. Keep the profile so their history is preserved.'; return }
  store.value.players = store.value.players.filter(p => p.id !== id)
  if (selectedPlayerId.value === id) selectedPlayerId.value = store.value.players[0].id
}
function startGame() {
  if (mode.value === 'solo') {
    const p = store.value.players.find(p => p.id === selectedPlayerId.value)
    if (!p) { message.value = 'Add a player profile first.'; return }
    gamePlayers.value = [p]
  } else {
    const list = selectedPlayers.value.map(id => store.value.players.find(p => p.id === id)).filter(Boolean)
    if (list.length < 2) { message.value = 'Select at least two players for multiplayer.'; return }
    gamePlayers.value = list
  }
  currentTurn.value = 0
  shots.value = []
  gameOver.value = false
  lastShot.value = null
  message.value = 'Drag from the bow towards the target, then release to shoot.'
  page.value = 'game'
  nextTick(resizeCanvas)
}
function resizeCanvas() {
  if (!canvasWrap.value) return
  const width = Math.max(320, Math.min(1100, canvasWrap.value.clientWidth))
  canvasWidth.value = width
  canvasHeight.value = Math.max(300, Math.min(480, width * 0.48))
  nextTick(drawScene)
}
function targetCenter() { return { x: canvasWidth.value - 145, y: canvasHeight.value * 0.48 } }
function bowPoint() { return { x: 105, y: canvasHeight.value * 0.68 } }
function drawScene() {
  const el = canvas.value
  if (!el) return
  const ctx = el.getContext('2d')
  const w = canvasWidth.value, h = canvasHeight.value
  ctx.clearRect(0,0,w,h)
  const sky = ctx.createLinearGradient(0,0,0,h)
  sky.addColorStop(0,'#d9eee7'); sky.addColorStop(1,'#edf3df')
  ctx.fillStyle = sky; ctx.fillRect(0,0,w,h)
  ctx.fillStyle = '#b8d2a4'; ctx.beginPath(); ctx.moveTo(0,h*.73); ctx.quadraticCurveTo(w*.28,h*.56,w*.52,h*.73); ctx.quadraticCurveTo(w*.75,h*.57,w,h*.70); ctx.lineTo(w,h); ctx.lineTo(0,h); ctx.fill()
  ctx.fillStyle = '#769c70'; ctx.fillRect(0,h*.82,w,h*.18)
  // range line
  ctx.strokeStyle = 'rgba(35,70,55,.24)'; ctx.setLineDash([7,8]); ctx.beginPath(); ctx.moveTo(140,h*.83); ctx.lineTo(w-145,h*.83); ctx.stroke(); ctx.setLineDash([])
  // target stand
  const t = targetCenter()
  ctx.strokeStyle = '#765b3f'; ctx.lineWidth = 8; ctx.beginPath(); ctx.moveTo(t.x,t.y+42); ctx.lineTo(t.x,t.y+130); ctx.stroke()
  ctx.fillStyle = '#fff'; ctx.beginPath(); ctx.arc(t.x,t.y,67,0,Math.PI*2); ctx.fill()
  const rings = [{r:59,c:'#e64c42'},{r:47,c:'#fff'},{r:35,c:'#e64c42'},{r:23,c:'#f5d44e'},{r:11,c:'#f5d44e'}]
  rings.forEach(r => { ctx.fillStyle=r.c; ctx.beginPath(); ctx.arc(t.x,t.y,r.r,0,Math.PI*2); ctx.fill() })
  ctx.strokeStyle='rgba(30,30,30,.35)'; ctx.lineWidth=1; ctx.beginPath(); ctx.arc(t.x,t.y,67,0,Math.PI*2); ctx.stroke()
  // bow
  const b = bowPoint()
  ctx.strokeStyle='#75492d'; ctx.lineWidth=7; ctx.beginPath(); ctx.arc(b.x+9,b.y,53,-Math.PI*.78,Math.PI*.78); ctx.stroke()
  ctx.strokeStyle='#f6f0d9'; ctx.lineWidth=2; ctx.beginPath(); ctx.moveTo(b.x-17,b.y-49); ctx.lineTo(b.x+28,b.y+49); ctx.stroke()
  ctx.fillStyle='#263c35'; ctx.font='600 13px system-ui'; ctx.fillText('YOUR BOW',b.x-27,b.y+77)
  // arrows already fired
  shots.value.forEach((shot,i) => {
    const p = { x: shot.x, y: shot.y }
    ctx.strokeStyle='#4a3426'; ctx.lineWidth=3; ctx.beginPath(); ctx.moveTo(p.x-19,p.y+7); ctx.lineTo(p.x+5,p.y-2); ctx.stroke()
    ctx.fillStyle='#263c35'; ctx.beginPath(); ctx.arc(p.x,p.y,3,0,Math.PI*2); ctx.fill()
  })
  if (dragging.value && dragPoint.value && aimPoint.value) {
    ctx.strokeStyle='rgba(30,57,48,.65)'; ctx.lineWidth=3; ctx.setLineDash([6,6]); ctx.beginPath(); ctx.moveTo(b.x,b.y); ctx.lineTo(aimPoint.value.x,aimPoint.value.y); ctx.stroke(); ctx.setLineDash([])
    ctx.fillStyle='#263c35'; ctx.beginPath(); ctx.arc(aimPoint.value.x,aimPoint.value.y,6,0,Math.PI*2); ctx.fill()
  }
}
function scoreToPoint(score, offset=0) {
  const t = targetCenter()
  const radii = {10:6,9:17,8:28,7:39,6:50,5:58,4:64,3:67,2:70,1:73,0:78}
  const r = radii[score] ?? 70
  const angle = offset * 2.399
  return { x: t.x + Math.cos(angle)*r, y: t.y + Math.sin(angle)*r }
}
function pointerPos(event) {
  const rect = canvas.value.getBoundingClientRect()
  return { x:(event.clientX-rect.left)*(canvasWidth.value/rect.width), y:(event.clientY-rect.top)*(canvasHeight.value/rect.height) }
}
function pointerDown(event) {
  if (gameOver.value || shots.value.length >= arrowLimit.value) return
  dragging.value = true; dragPoint.value = pointerPos(event); aimPoint.value = dragPoint.value
  canvas.value.setPointerCapture?.(event.pointerId)
  drawScene()
}
function pointerMove(event) {
  if (!dragging.value) return
  aimPoint.value = pointerPos(event); drawScene()
}
function pointerUp(event) {
  if (!dragging.value) return
  aimPoint.value = pointerPos(event); dragging.value = false
  const bow = bowPoint(), aim = aimPoint.value
  const dx = bow.x - aim.x, dy = bow.y - aim.y
  const pull = Math.hypot(dx,dy)
  if (pull < 15) { message.value = 'Pull the arrow back further before releasing.'; drawScene(); return }
  const target = targetCenter()
  const strength = Math.min(1, pull / 230)
  const direction = Math.atan2(aim.y-bow.y, aim.x-bow.x)
  // Mouse drag direction is reversed into a forward shot. Map aim to a plausible target landing point.
  const targetX = target.x + (aim.x - bow.x) * 0.24 + (Math.random()-.5)*(1-strength)*38
  const targetY = target.y + (aim.y - bow.y) * 0.22 + (Math.random()-.5)*(1-strength)*38
  const radius = Math.hypot(targetX-target.x,targetY-target.y)
  const score = radius <= 8 ? 10 : radius <= 19 ? 9 : radius <= 30 ? 8 : radius <= 41 ? 7 : radius <= 52 ? 6 : radius <= 61 ? 5 : radius <= 67 ? 4 : radius <= 73 ? 3 : radius <= 79 ? 2 : radius <= 86 ? 1 : 0
  const offset = shots.value.length + (currentTurn.value*13)
  const shot = { score, offset, x:targetX, y:targetY }
  shots.value.push(shot); lastShot.value = shot
  message.value = score === 10 ? 'Bullseye! 10 points.' : `Arrow scored ${score} point${score===1?'':'s'}.`
  if (shots.value.length >= arrowLimit.value) finishTurn()
  drawScene()
}
function finishTurn() {
  if (mode.value === 'multiplayer' && currentTurn.value < gamePlayers.value.length-1) {
    gameOver.value = true
    message.value = `${activePlayer.value.name} finished. Pass the device to ${gamePlayers.value[currentTurn.value+1].name}.`
  } else {
    gameOver.value = true
    message.value = 'Round complete! Save your result.'
  }
}
function nextPlayerTurn() {
  currentTurn.value++
  shots.value = []; gameOver.value = false; lastShot.value = null
  message.value = `${activePlayer.value.name}, it is your turn. Drag back and release to shoot.`
  drawScene()
}
function saveResults() {
  const date = new Date().toISOString()
  const row = { id:store.value.nextGameId++, playerId:activePlayer.value.id, playerName:activePlayer.value.name, mode:mode.value, round:roundType.value==='practice'?'Practice':'6-arrow Challenge', distance:distance.value, score:totalScore.value, arrows:shots.value.length, date }
  store.value.history.push(row)
  if (mode.value==='multiplayer' && currentTurn.value < gamePlayers.value.length-1) { nextPlayerTurn(); return }
  page.value='results'
}
function saveAllMultiplayerResults() {
  // In pass-and-play each player's score is saved before moving to the next player.
  const date = new Date().toISOString()
  store.value.history.push({ id:store.value.nextGameId++, playerId:activePlayer.value.id, playerName:activePlayer.value.name, mode:mode.value, round:roundType.value==='practice'?'Practice':'6-arrow Challenge', distance:distance.value, score:totalScore.value, arrows:shots.value.length, date })
  if (currentTurn.value < gamePlayers.value.length-1) nextPlayerTurn()
  else page.value='results'
}
function abandonGame() { if (confirm('Leave this game? Unsaved arrows will be lost.')) page.value='home' }
function formatDate(value) { return new Date(value).toLocaleDateString(undefined,{day:'numeric',month:'short',year:'numeric'}) }
function newGame() { page.value='setup'; shots.value=[]; gameOver.value=false }
function resetDemo() {
  if (!confirm('Reset all locally saved players and scores?')) return
  localStorage.removeItem(STORE_KEY); store.value=structuredClone(initial); selectedPlayerId.value=1; selectedPlayers.value=[]; page.value='home'
}
watch([shots,dragging,canvasWidth,canvasHeight], () => nextTick(drawScene), { deep:true })
onMounted(() => { window.addEventListener('resize',resizeCanvas); nextTick(resizeCanvas) })
</script>

<template>
  <div class="app-shell">
    <header class="topbar">
      <button class="brand" @click="page='home'"><span class="brand-mark">➶</span><span>ARROW<span class="brand-light">TRACK</span><small>ARCHERY GAME & SCORE TRACKER</small></span></button>
      <nav><button :class="{active:page==='home'}" @click="page='home'">Dashboard</button><button :class="{active:page==='setup'}" @click="newGame">Play</button><button :class="{active:page==='leaderboard'}" @click="page='leaderboard'">Leaderboard</button><button :class="{active:page==='history'}" @click="page='history'">Score history</button></nav>
      <div class="top-user"><span class="avatar">{{ (store.players[0]?.name || 'A').slice(0,1).toUpperCase() }}</span><span>{{ store.players[0]?.name || 'Player' }}</span></div>
    </header>

    <main class="main-wrap">
      <section v-if="page==='home'" class="page-enter">
        <div class="hero-panel">
          <div><p class="eyebrow">WELCOME TO YOUR RANGE</p><h1>Take your shot.<br><em>Beat your best.</em></h1><p class="hero-copy">A little practice, a little competition. Play solo or challenge friends on the same screen.</p><div class="hero-actions"><button class="btn-primary" @click="newGame">➶ &nbsp; Start playing</button><button class="btn-ghost" @click="page='leaderboard'">View leaderboard ↗</button></div></div>
          <div class="hero-art"><div class="sun"></div><div class="hill hill-one"></div><div class="hill hill-two"></div><div class="target target-large"><i></i><i></i><i></i><i></i><b></b></div><div class="hero-arrow">⟶</div><div class="hero-caption">AIM · RELEASE · REPEAT</div></div>
        </div>
        <div class="section-heading"><div><p class="eyebrow">YOUR PROGRESS</p><h2>Dashboard overview</h2></div><button class="text-link" @click="page='history'">All activity ↗</button></div>
        <div class="stats-grid"><article class="stat-card"><span class="stat-icon">◎</span><span class="stat-label">Games played</span><strong>{{ store.history.length }}</strong><small>Saved sessions</small></article><article class="stat-card"><span class="stat-icon">✦</span><span class="stat-label">Players</span><strong>{{ store.players.length }}</strong><small>Local profiles</small></article><article class="stat-card"><span class="stat-icon">↗</span><span class="stat-label">Top score</span><strong>{{ Math.max(0,...store.history.map(x=>x.score)) }}</strong><small>Across all players</small></article><article class="stat-card"><span class="stat-icon">♛</span><span class="stat-label">Personal best</span><strong>{{ personalBest(selectedPlayerId) }}</strong><small>{{ playerById(selectedPlayerId) }}</small></article></div>
        <div class="content-grid"><section class="panel"><div class="panel-head"><div><p class="eyebrow">RECENT ACTIVITY</p><h3>Latest games</h3></div><button class="text-link" @click="page='history'">View all</button></div><div v-if="store.history.length" class="table-scroll"><table><thead><tr><th>PLAYER</th><th>MODE</th><th>DATE</th><th>SCORE</th></tr></thead><tbody><tr v-for="item in [...store.history].slice(-5).reverse()" :key="item.id"><td><b>{{ item.playerName }}</b></td><td><span class="mode-pill">{{ item.mode }}</span></td><td>{{ formatDate(item.date) }}</td><td class="score-cell">{{ item.score }}</td></tr></tbody></table></div><div v-else class="empty-state"><span>◎</span><b>No games yet</b><p>Your completed games will appear here.</p><button class="btn-primary small" @click="newGame">Play your first round</button></div></section>
          <section class="panel players-panel"><div class="panel-head"><div><p class="eyebrow">PLAYER PROFILES</p><h3>Your players</h3></div><button class="icon-button" @click="showAddPlayer=true">＋</button></div><div v-for="(player,index) in store.players" :key="player.id" class="player-row"><span class="player-avatar" :class="'avatar-'+(index%4)">{{ player.name.slice(0,1).toUpperCase() }}</span><span class="player-info"><b>{{ player.name }}</b><small>Personal best: {{ personalBest(player.id) }}</small></span><button class="tiny-link" @click="selectedPlayerId=player.id;newGame()">Play</button><button class="remove-link" title="Remove profile" @click="removePlayer(player.id)">×</button></div><button class="add-player-button" @click="showAddPlayer=true">＋ Add another player</button></section></div>
      </section>

      <section v-else-if="page==='setup'" class="setup-page page-enter">
        <button class="back-link" @click="page='home'">← Back to dashboard</button><div class="page-title"><p class="eyebrow">READY, ARCHER?</p><h1>Set up your game</h1><p>Choose how you want to play and step up to the line.</p></div>
        <div class="setup-card"><div class="stepper"><span class="step active"><b>1</b> Game setup</span><i></i><span class="step"><b>2</b> Shoot</span><i></i><span class="step"><b>3</b> Results</span></div>
          <div class="setup-section"><h3>Choose your game mode</h3><div class="mode-options"><button :class="['mode-option',{chosen:mode==='solo'}]" @click="mode='solo'"><span class="mode-symbol">◎</span><b>Solo practice</b><small>Play at your own pace and beat your personal best.</small></button><button :class="['mode-option',{chosen:mode==='multiplayer'}]" @click="mode='multiplayer'"><span class="mode-symbol">♧</span><b>Pass & play</b><small>Take turns with friends on the same device.</small></button></div></div>
          <div class="setup-section"><h3>{{ mode==='solo'?'Who is playing?':'Choose your players' }}</h3><div v-if="mode==='solo'" class="select-wrap"><select v-model.number="selectedPlayerId"><option v-for="player in store.players" :key="player.id" :value="player.id">{{ player.name }}</option></select><span>⌄</span></div><div v-else class="player-select-grid"><label v-for="player in store.players" :key="player.id" class="check-player"><input type="checkbox" :value="player.id" v-model="selectedPlayers"><span class="fake-check">✓</span><b>{{ player.name }}</b></label></div><button class="inline-add" @click="showAddPlayer=true">＋ Add a new player profile</button></div>
          <div class="setup-section"><h3>Choose your round</h3><div class="round-options"><button :class="['round-option',{chosen:roundType==='practice'}]" @click="roundType='practice'"><span class="round-tag">FREE SHOOT</span><b>Practice session</b><small>12 arrows · No pressure</small></button><button :class="['round-option',{chosen:roundType==='challenge'}]" @click="roundType='challenge'"><span class="round-tag">SCORE ATTACK</span><b>6-arrow challenge</b><small>6 arrows · Beat the clock? No, beat your best.</small></button></div></div>
          <div class="setup-section distance-section"><div><h3>Target distance</h3><p>Choose how far away the target is.</p></div><div class="distance-buttons"><button v-for="d in [10,18,30,50,70]" :key="d" :class="{chosen:distance===d}" @click="distance=d">{{ d }}m</button></div></div>
          <div class="setup-footer"><span>✦ &nbsp; {{ arrowLimit }} arrows per player</span><button class="btn-primary" @click="startGame">Go to shooting line &nbsp; →</button></div>
        </div>
      </section>

      <section v-else-if="page==='game'" class="game-page page-enter">
        <div class="game-topline"><button class="back-link" @click="abandonGame">← Leave game</button><span class="game-mode-label">{{ mode==='solo'?'SOLO PRACTICE':'PASS & PLAY' }} · {{ distance }} METRES</span></div>
        <div class="game-heading"><div><p class="eyebrow">{{ mode==='solo'?'YOUR SESSION':'PLAYER '+(currentTurn+1)+' OF '+gamePlayers.length }}</p><h1>{{ activePlayer?.name }}<span>'s turn</span></h1></div><div class="score-live"><small>ROUND SCORE</small><strong>{{ totalScore }}</strong></div></div>
        <div class="game-layout"><div class="range-column"><div class="range-toolbar"><span><i class="live-dot"></i> LIVE RANGE</span><span>{{ shots.length }} / {{ arrowLimit }} ARROWS</span></div><div class="canvas-wrap" ref="canvasWrap"><canvas ref="canvas" :width="canvasWidth" :height="canvasHeight" @pointerdown="pointerDown" @pointermove="pointerMove" @pointerup="pointerUp" @pointercancel="pointerUp"></canvas><div class="canvas-hint" v-if="shots.length===0 && !gameOver"><span>↖</span> DRAG BACK & RELEASE TO SHOOT</div><div v-if="gameOver" class="game-overlay"><div class="overlay-card"><span class="overlay-icon">✦</span><h2>{{ mode==='multiplayer' && currentTurn<gamePlayers.length-1 ? 'Pass the device' : 'Round complete!' }}</h2><p>{{ message }}</p><button v-if="mode==='multiplayer' && currentTurn<gamePlayers.length-1" class="btn-primary" @click="saveAllMultiplayerResults">Ready for next player →</button><button v-else class="btn-primary" @click="saveResults">Save my result →</button></div></div></div><div class="game-message"><span>✦</span>{{ message }}</div><div class="game-controls"><div><small>HOW TO SHOOT</small><b>Click and drag from the bow backwards, then release.</b></div><button class="btn-outline" :disabled="!shots.length || gameOver" @click="shots.pop();message='Last arrow removed. Take another shot.'">↶ Undo last</button></div></div>
          <aside class="score-column"><div class="score-panel"><p class="eyebrow">YOUR SCORECARD</p><div class="score-total"><strong>{{ totalScore }}</strong><span>POINTS</span></div><div class="score-metrics"><div><b>{{ shots.length }}</b><small>Arrows shot</small></div><div><b>{{ arrowsLeft }}</b><small>Remaining</small></div><div><b>{{ averageScore }}</b><small>Average</small></div></div><div class="arrow-slots"><span v-for="n in arrowLimit" :key="n" :class="{filled:shots[n-1]}" :title="shots[n-1] ? `Arrow ${n}: ${shots[n-1].score}` : `Arrow ${n}`">{{ shots[n-1]?.score ?? '—' }}</span></div><div class="shot-list"><div v-for="(shot,index) in shots" :key="index"><span>Arrow {{ index+1 }}</span><b>{{ shot.score }} pts</b></div></div></div><div class="tip-panel"><span>☼</span><div><b>Quick tip</b><p>Pull further back for a steadier shot. Small movements make a big difference near the centre.</p></div></div><div v-if="mode==='multiplayer'" class="turn-panel"><b>Players in this game</b><div v-for="(p,index) in gamePlayers" :key="p.id" :class="['turn-player',{current:index===currentTurn}]"><span>{{ p.name }}</span><small>{{ index===currentTurn?'PLAYING':index<currentTurn?'DONE':'UP NEXT' }}</small></div></div></aside></div>
      </section>

      <section v-else-if="page==='results'" class="results-page page-enter"><div class="result-hero"><span class="result-star">✦</span><p class="eyebrow">SESSION SAVED</p><h1>Nice shooting, {{ gamePlayers[0]?.name?.split(' ')[0] || 'archer' }}.</h1><p>Your result has been added to your score history.</p><div class="result-score"><strong>{{ store.history[store.history.length-1]?.score ?? totalScore }}</strong><span>POINTS</span></div><div class="result-stats"><div><b>{{ store.history[store.history.length-1]?.arrows ?? shots.length }}</b><small>Arrows</small></div><div><b>{{ distance }}m</b><small>Distance</small></div><div><b>{{ roundType==='practice'?'Practice':'Challenge' }}</b><small>Round type</small></div></div></div><div class="result-actions"><button class="btn-primary" @click="newGame">Play again</button><button class="btn-outline" @click="page='history'">View score history</button><button class="text-link" @click="page='home'">Back to dashboard</button></div></section>

      <section v-else-if="page==='history'" class="page-enter"><div class="page-title"><p class="eyebrow">YOUR PROGRESS</p><h1>Score history</h1><p>Every saved session, all in one place.</p></div><div class="panel history-panel"><div class="filter-tabs"><button :class="{active:selectedTab==='all'}" @click="selectedTab='all'">All games</button><button :class="{active:selectedTab==='solo'}" @click="selectedTab='solo'">Solo</button><button :class="{active:selectedTab==='multiplayer'}" @click="selectedTab='multiplayer'">Pass & play</button></div><div v-if="filteredHistory.length" class="table-scroll"><table><thead><tr><th>PLAYER</th><th>GAME MODE</th><th>ROUND</th><th>DISTANCE</th><th>DATE</th><th>SCORE</th></tr></thead><tbody><tr v-for="item in filteredHistory" :key="item.id"><td><b>{{ item.playerName }}</b></td><td>{{ item.mode }}</td><td>{{ item.round }}</td><td>{{ item.distance }}m</td><td>{{ formatDate(item.date) }}</td><td class="score-cell">{{ item.score }}</td></tr></tbody></table></div><div v-else class="empty-state"><span>◎</span><b>No saved games found</b><p>Play a round to start building your history.</p><button class="btn-primary small" @click="newGame">Start playing</button></div></div></section>

      <section v-else-if="page==='leaderboard'" class="page-enter"><div class="page-title"><p class="eyebrow">FRIENDLY COMPETITION</p><h1>Leaderboard</h1><p>See who's hitting the highest scores.</p></div><div class="leaderboard-panel"><div class="leaderboard-head"><span>RANK</span><span>PLAYER</span><span>GAMES</span><span>PERSONAL BEST</span></div><div v-for="(player,index) in sortedPlayers" :key="player.id" class="leaderboard-row"><span class="rank-number" :class="{'rank-gold':index===0}">{{ String(index+1).padStart(2,'0') }}</span><span class="leader-player"><i>{{ player.name.slice(0,1).toUpperCase() }}</i><b>{{ player.name }}</b></span><span>{{ store.history.filter(h=>h.playerId===player.id).length }}</span><strong>{{ personalBest(player.id) }}</strong></div><div v-if="!store.history.length" class="empty-state"><span>♛</span><b>Leaderboard is waiting</b><p>Complete a game to claim the top spot.</p></div></div></section>
    </main>

    <footer class="footer"><span>➶ ARROWTRACK</span><span>Practice makes progress. &nbsp;·&nbsp; Demo version — data saved on this device.</span><button @click="resetDemo">Reset demo data</button></footer>

    <div v-if="showAddPlayer" class="modal-backdrop" @click.self="showAddPlayer=false"><div class="modal"><button class="modal-close" @click="showAddPlayer=false">×</button><p class="eyebrow">NEW ARCHER</p><h2>Add a player</h2><p>Create a profile so they can play solo or join a pass-and-play game.</p><label class="field-label">PLAYER NAME<input v-model="newPlayerName" autofocus maxlength="30" placeholder="e.g. Taylor Smith" @keyup.enter="addPlayer"></label><div class="modal-actions"><button class="btn-outline" @click="showAddPlayer=false">Cancel</button><button class="btn-primary" :disabled="!newPlayerName.trim()" @click="addPlayer">Save player</button></div></div></div>
  </div>
</template>
