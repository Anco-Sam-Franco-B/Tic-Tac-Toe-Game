<script setup>
import { ref, computed, watch, onMounted } from "vue"
import clickedSound from "./assets/ting.mp3"
import bgMusic from "./assets/music.mp3"
import confettiSoundFile from "./assets/confetti.mp3"

// ---------- GAME STATE ----------
const player = ref("X")
const board = ref([
  ["","",""],
  ["","",""],
  ["","",""]
])
const history = ref([])
const winLine = ref([])

// ---------- PLAYER INFO ----------
const playerXName = ref("Player X")
const playerOName = ref("Player O")
const mode = ref("HvAI") // default Human vs AI

// ---------- SCORE ----------
const scoreX = ref(0)
const scoreO = ref(0)

// ---------- SOUND ----------
let music = null
let confettiSound = null
const musicPlaying = ref(false)

// Load saved scores, names, mode
onMounted(()=>{
  const savedScore = JSON.parse(localStorage.getItem("ttt-score"))
  const savedNames = JSON.parse(localStorage.getItem("ttt-names"))
  const savedMode = localStorage.getItem("ttt-mode")
  if(savedScore){
    scoreX.value = savedScore.X
    scoreO.value = savedScore.O
  }
  if(savedNames){
    playerXName.value = savedNames.X
    playerOName.value = savedNames.O
  }
  if(savedMode) mode.value = savedMode

  // Background music
  music = new Audio(bgMusic)
  music.loop = true
  music.volume = 0.5
  confettiSound = new Audio(confettiSoundFile)

  document.addEventListener("click", playMusic, { once:true })
})

// Save scores and names
watch([scoreX,scoreO],()=>{ localStorage.setItem("ttt-score", JSON.stringify({X:scoreX.value,O:scoreO.value})) })
watch([playerXName,playerOName],()=>{ localStorage.setItem("ttt-names", JSON.stringify({X:playerXName.value,O:playerOName.value})) })
watch(mode,()=>{ localStorage.setItem("ttt-mode", mode.value) })

// ---------- WIN LOGIC ----------
const lines = [
  [0,1,2],[3,4,5],[6,7,8],
  [0,3,6],[1,4,7],[2,5,8],
  [0,4,8],[2,4,6]
]
function calculateWinner(squares){
  for(let i=0;i<lines.length;i++){
    const [a,b,c] = lines[i]
    if(squares[a] && squares[a]===squares[b] && squares[a]===squares[c]){
      winLine.value = [a,b,c]
      return squares[a]
    }
  }
  winLine.value=[]
  return null
}
const flatBoard = computed(()=> board.value.flat())
const winner = computed(()=> calculateWinner(flatBoard.value))

// ---------- DRAW DETECTION ----------
const draw = computed(()=> !winner.value && flatBoard.value.every(c=>c!==""))

// ---------- DETERMINE IF AI SHOULD MOVE ----------
const isAIturn = computed(()=>{
  return (mode.value==="HvAI" && player.value==="O") || mode.value==="AIvAI"
})

// ---------- MOVE ----------
function MakeMove(x,y){
  if(winner.value || board.value[x][y]!=="") return
  board.value[x][y]=player.value
  history.value.push({player:player.value, position:[x,y]})
  playSound()
  player.value = player.value==="X" ? "O":"X"

  if(isAIturn.value && !winner.value){
    setTimeout(AIMove,300)
  }
}

// ---------- SIMPLE AI ----------
function AIMove(){
  const empty=[]
  board.value.forEach((row,i)=>{ row.forEach((cell,j)=>{ if(cell==="") empty.push([i,j]) }) })
  if(empty.length===0) return
  const move = empty[Math.floor(Math.random()*empty.length)]
  MakeMove(move[0],move[1])
}

// ---------- SCORE UPDATE ----------
watch(winner,(w)=>{
  if(!w) return
  if(w==="X") scoreX.value++
  if(w==="O") scoreO.value++
  playConfetti()
})

// ---------- RESET ----------
function ResetGame(){ board.value=[["","",""],["","",""],["","",""]]; history.value=[]; winLine.value=[]; player.value="X" }
function ResetScores(){ scoreX.value=0; scoreO.value=0 }

// ---------- SOUND FUNCTIONS ----------
function playSound(){ const audio = new Audio(clickedSound); audio.play() }
function playMusic(){ if(!music) return; music.play().then(()=>musicPlaying.value=true).catch(()=>{}) }
function playConfetti(){ if(confettiSound) confettiSound.play(); confetti() }

// ---------- CONFETTI EFFECT ----------
function confetti(){
  const duration = 3000
  const end = Date.now() + duration
  const colors = ['#ff4da6','#4da6ff','#00ffff','#f39c12','#e74c3c']
  const canvas = document.createElement('canvas')
  document.body.appendChild(canvas)
  canvas.style.position='fixed'
  canvas.style.top='0'
  canvas.style.left='0'
  canvas.style.width='100%'
  canvas.style.height='100%'
  canvas.style.pointerEvents='none'
  const ctx = canvas.getContext('2d')
  const w = canvas.width = window.innerWidth
  const h = canvas.height = window.innerHeight
  const particles = Array.from({length:150}).map(()=>({
    x: Math.random()*w,
    y: Math.random()*h - h,
    r: Math.random()*6+4,
    d: Math.random()*15,
    color: colors[Math.floor(Math.random()*colors.length)],
    tilt: Math.random()*10-10,
    tiltAngle: 0
  }))
  function draw(){
    ctx.clearRect(0,0,w,h)
    particles.forEach(p=>{
      ctx.beginPath()
      ctx.lineWidth = p.r
      ctx.strokeStyle = p.color
      ctx.moveTo(p.x + p.tilt + p.r/2, p.y)
      ctx.lineTo(p.x + p.tilt, p.y + p.tilt + p.r/2)
      ctx.stroke()
    })
    update()
  }
  function update(){
    particles.forEach(p=>{
      p.tiltAngle += 0.1
      p.y += (Math.cos(p.d) + 3 + p.r/2)/2
      p.tilt = Math.sin(p.tiltAngle) * 15
      if(p.y>h){ p.x=Math.random()*w; p.y=-20 }
    })
  }
  (function animloop(){ draw(); if(Date.now()<end) requestAnimationFrame(animloop); else document.body.removeChild(canvas) })()
}

// ---------- LASER LINE ----------
const laserClass = computed(()=>{
  if(!winLine.value.length) return ""
  const [a,b,c] = winLine.value
  const map = {
    "0,1,2":"laser-row1","3,4,5":"laser-row2","6,7,8":"laser-row3",
    "0,3,6":"laser-col1","1,4,7":"laser-col2","2,5,8":"laser-col3",
    "0,4,8":"laser-diag1","2,4,6":"laser-diag2"
  }
  return map[[a,b,c].join(",")]
})
</script>

<template>
  <div class="container">
    <h1 class="title">Anco Sam Franco Tech Copyright $copy; {{ new Date.getFullYear() }}. All right are reserved</h1>
    <h1 class="title">Tic Tac Toe</h1>

    <!-- PLAYER NAMES -->
    <div class="player-inputs">
      <input v-model="playerXName" placeholder="Player X Name" />
      <input v-model="playerOName" placeholder="Player O Name" />
    </div>

    <!-- MODE SELECT -->
    <div class="mode-select">
      <label><input type="radio" value="HvH" v-model="mode" /> Human vs Human</label>
      <label><input type="radio" value="HvAI" v-model="mode" /> Human vs AI</label>
      <label><input type="radio" value="AIvAI" v-model="mode" /> AI vs AI</label>
    </div>

    <!-- SCOREBOARD -->
    <div class="scoreboard">
      <div class="score-card x">{{playerXName}} : {{scoreX}}</div>
      <div class="score-card o">{{playerOName}} : {{scoreO}}</div>
    </div>

    <!-- STATUS -->
    <h2 class="status" v-if="!winner && !draw">Turn: {{player==="X"?playerXName:playerOName}}</h2>
    <h2 class="status win-text" v-if="winner">Winner: {{winner==="X"?playerXName:playerOName}}</h2>
    <h2 class="status draw-text" v-if="draw">Draw Game</h2>

    <!-- BOARD -->
    <div class="board">
      <div v-for="(row,x) in board" :key="x" class="row">
        <div v-for="(cell,y) in row" :key="y" class="cell" @click="MakeMove(x,y)" :class="cell.toLowerCase()">{{cell}}</div>
      </div>
      <div v-if="winner" class="laser" :class="laserClass"></div>
    </div>

    <!-- BUTTONS -->
    <div class="buttons">
      <button class="btn-reset" @click="ResetGame">New Game</button>
      <button class="btn-reset reset-scores" @click="ResetScores">Reset Scores</button>
    </div>

    <!-- HISTORY -->
    <h3 class="history-title">History</h3>
    <ul class="history">
      <li v-for="(h,i) in history" :key="i">
        Move {{i+1}} : {{h.player==="X"?playerXName:playerOName}} ({{h.position[0]}},{{h.position[1]}})
      </li>
    </ul>

  </div>
</template>

<style>
/* Container & Typography */
body{margin:0;font-family:'Segoe UI',sans-serif;background:linear-gradient(135deg,#0f2027,#203a43,#2c5364);color:white}
.container{max-width:450px;margin:20px auto;padding:10px;display:flex;flex-direction:column;align-items:center;gap:10px}
.title{font-size:2rem;font-weight:600;text-align:center}
.player-inputs{display:flex;gap:10px;width:100%;margin-bottom:10px}
.player-inputs input{flex:1;padding:8px;border-radius:10px;border:none;background: rgba(255,255,255,0.1);color:white;text-align:center}
.mode-select{display:flex;gap:10px;justify-content:center;margin-bottom:10px;font-size:.9rem}
.mode-select label{cursor:pointer}
.scoreboard{display:flex;gap:10px;width:100%;margin-bottom:10px}
.score-card{flex:1;padding:10px 0;text-align:center;border-radius:15px;background: rgba(255,255,255,0.1);backdrop-filter: blur(5px);box-shadow:0 4px 15px rgba(0,0,0,0.2)}
.score-card.x{color:#ff4da6}
.score-card.o{color:#4da6ff}
.board{display:flex;flex-direction:column;gap:10px;width:100%;aspect-ratio:1/1;background: rgba(255,255,255,0.05);border-radius:20px;padding:10px;box-shadow:0 8px 20px rgba(0,0,0,0.4);position: relative}
.row{display:flex;flex:1;gap:10px}
.cell{flex:1;display:flex;justify-content:center;align-items:center;font-size:2.5rem;cursor:pointer;border-radius:15px;background: rgba(255,255,255,0.05);box-shadow:0 4px 15px rgba(0,0,0,0.3);transition:all .3s}
.cell:hover{background: rgba(255,255,255,0.15);transform: scale(1.05)}
.cell.x{color:#ff4da6;text-shadow:0 0 8px #ff4da6}
.cell.o{color:#4da6ff;text-shadow:0 0 8px #4da6ff}
.status{font-size:1.2rem}
.win-text{color:#00ffff;text-shadow:0 0 8px #00ffff}
.draw-text{color:#f39c12}
.buttons{display:flex;gap:10px;margin-top:10px;width:100%}
.btn-reset{flex:1;padding:10px 15px;border:none;border-radius:15px;background:linear-gradient(45deg,#ff4da6,#4da6ff);color:white;cursor:pointer;transition:transform .2s}
.btn-reset:hover{transform: scale(1.05)}
.reset-scores{background:linear-gradient(45deg,#f39c12,#e74c3c)}
.history-title{margin-top:15px;font-weight:bold}
.history{max-height:120px;width:100%;overflow:auto;font-size:.9rem;background: rgba(255,255,255,0.05);border-radius:10px;padding:5px 10px}

/* LASER LINES */
.laser{position:absolute;background: linear-gradient(90deg, transparent, #00ffff, #ffffff, #00ffff, transparent);box-shadow:0 0 10px #00ffff,0 0 20px #00ffff,0 0 40px #00ffff;animation: laserGlow .7s infinite alternate}
@keyframes laserGlow{from{opacity:.6} to{opacity:1}}
.laser-row1{top:13%; left:0; width:100%; height:6px}
.laser-row2{top:50%; left:0; width:100%; height:6px}
.laser-row3{bottom:13%; left:0; width:100%; height:6px}
.laser-col1{left:13%; top:0; width:6px; height:100%}
.laser-col2{left:50%; top:0; width:6px; height:100%}
.laser-col3{right:13%; top:0; width:6px; height:100%}
.laser-diag1 {
  width: 100%;
  height: 6px;
  top: 50%;
  left: 0;
  transform: rotate(45deg);
  transform-origin: center;
}

.laser-diag2 {
  width: 100%;
  height: 6px;
  top: 50%;
  left: 0;
  transform: rotate(-45deg);
  transform-origin: center;
}
</style>
