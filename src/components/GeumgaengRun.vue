<script setup>
import { computed, onBeforeUnmount, onMounted, ref } from 'vue';
import { api } from '../lib/api';

/**
 * 금갱런 — 금갱이를 연속 5번 두드리면 열리는 미니게임(크롬 공룡런 스타일).
 * 스페이스바 / 위 화살표 / 클릭·탭으로 점프해서 장애물을 피한다. 공중에서 한 번 더 눌러 2단 점프도 가능하다.
 *
 * 기록은 서버(run_scores)에 남아 모든 방문자가 같은 명예의 전당을 본다.
 * 사람(uid)당 최고 기록 한 줄만 남기므로 한 사람이 시상대를 독식하지 않는다.
 */

const emit = defineEmits(['close']);

const canvasRef = ref(null);
const score = ref(0);
const highScore = ref(Number(localStorage.getItem('geumgaengrun-highscore')) || 0);
const status = ref('ready'); // ready | playing | over

/* ------------------------------------------------------ 명예의 전당 */

const NAME_KEY = 'geumgaengrun-name';
const NAME_MAX = 12;

const board = ref([]);
const boardError = ref('');
const playerName = ref(localStorage.getItem(NAME_KEY) || '');
const saving = ref(false);
const saveError = ref('');
/** 이번 판을 등록하고 받은 등수. null 이면 아직 등록 전이다. */
const myRank = ref(null);
const savedScore = ref(null);
const myBest = ref(null);
const improved = ref(false);

const podium = computed(() => board.value.slice(0, 3));
const runners = computed(() => board.value.slice(3));

async function loadBoard() {
  try {
    const { scores, mine } = await api.runScores();
    board.value = scores;
    // 전에 쓰던 이름이 서버에 있으면 채워 준다(이 브라우저에 기록이 없을 때).
    if (mine?.name && !playerName.value) playerName.value = mine.name;
    boardError.value = '';
  } catch (e) {
    boardError.value = e.offline ? '명예의 전당을 불러오지 못했어요. (서버 연결 없음)' : e.message;
  }
}

async function submitScore() {
  const name = playerName.value.trim().slice(0, NAME_MAX);
  if (!name) {
    saveError.value = '이름을 입력해 주세요.';
    return;
  }
  saving.value = true;
  saveError.value = '';
  try {
    const res = await api.saveRunScore(name, score.value);
    localStorage.setItem(NAME_KEY, name);
    board.value = res.scores;
    myRank.value = res.rank;
    myBest.value = res.best;
    improved.value = res.improved;
    savedScore.value = score.value;
  } catch (e) {
    saveError.value = e.message;
  } finally {
    saving.value = false;
  }
}

const GROUND_Y = 150;
const GRAVITY = 0.0022;
const JUMP_VELOCITY = -0.62;
const PLAYER_X = 46;
const PLAYER_W = 46;
const PLAYER_H = 56;
const FRAME_W = 208;
const FRAME_H = 260;
const FRAME_COUNT = 8;

/** 착지 전까지 쓸 수 있는 점프 횟수(2단 점프). */
const MAX_JUMPS = 2;
const player = { y: GROUND_Y - PLAYER_H, vy: 0, jumpCount: 0 };
let obstacles = [];
let speed = 0.28;
let elapsed = 0;
let spawnTimer = 0;
let nextSpawnGap = 700;

let ctx;
let raf = 0;
let lastTime = 0;

const sprite = new Image();
sprite.src = new URL('../assets/geumgaengi-typing.webp', import.meta.url).href;

function reset() {
  player.y = GROUND_Y - PLAYER_H;
  player.vy = 0;
  player.jumpCount = 0;
  obstacles = [];
  speed = 0.28;
  elapsed = 0;
  spawnTimer = 0;
  nextSpawnGap = 700 + Math.random() * 500;
  score.value = 0;
}

function start() {
  status.value = 'playing';
  myRank.value = null;
  savedScore.value = null;
  myBest.value = null;
  saveError.value = '';
  reset();
  lastTime = performance.now();
  cancelAnimationFrame(raf);
  raf = requestAnimationFrame(loop);
}

function jump() {
  if (status.value !== 'playing') {
    start();
    return;
  }
  if (player.jumpCount < MAX_JUMPS) {
    player.vy = JUMP_VELOCITY;
    player.jumpCount += 1;
  }
}

function endGame() {
  status.value = 'over';
  cancelAnimationFrame(raf);
  const final = Math.floor(elapsed / 100);
  score.value = final;
  if (final > highScore.value) {
    highScore.value = final;
    localStorage.setItem('geumgaengrun-highscore', String(final));
  }
  draw();
}

function loop(now) {
  const dt = Math.min(now - lastTime, 40);
  lastTime = now;
  elapsed += dt;

  player.vy += GRAVITY * dt;
  player.y += player.vy * dt;
  if (player.y >= GROUND_Y - PLAYER_H) {
    player.y = GROUND_Y - PLAYER_H;
    player.vy = 0;
    player.jumpCount = 0;
  }

  speed = 0.28 + Math.min(elapsed / 20000, 0.35);

  spawnTimer += dt;
  if (spawnTimer >= nextSpawnGap) {
    spawnTimer = 0;
    nextSpawnGap = (500 + Math.random() * 700) / (speed / 0.28);
    const h = 28 + Math.random() * 26;
    obstacles.push({ x: canvasRef.value.width, w: 16 + Math.random() * 12, h });
  }

  obstacles.forEach((o) => (o.x -= speed * dt));
  obstacles = obstacles.filter((o) => o.x + o.w > 0);

  const pRect = { x: PLAYER_X + 8, y: player.y + 8, w: PLAYER_W - 16, h: PLAYER_H - 12 };
  for (const o of obstacles) {
    const oRect = { x: o.x, y: GROUND_Y - o.h, w: o.w, h: o.h };
    if (
      pRect.x < oRect.x + oRect.w &&
      pRect.x + pRect.w > oRect.x &&
      pRect.y < oRect.y + oRect.h &&
      pRect.y + pRect.h > oRect.y
    ) {
      endGame();
      return;
    }
  }

  score.value = Math.floor(elapsed / 100);
  draw();
  raf = requestAnimationFrame(loop);
}

function draw() {
  const canvas = canvasRef.value;
  if (!canvas || !ctx) return;
  const w = canvas.width;
  const h = canvas.height;
  ctx.clearRect(0, 0, w, h);

  ctx.strokeStyle = '#c9c2b6';
  ctx.lineWidth = 2;
  ctx.beginPath();
  ctx.moveTo(0, GROUND_Y);
  ctx.lineTo(w, GROUND_Y);
  ctx.stroke();

  ctx.fillStyle = '#2f6b4f';
  obstacles.forEach((o) => {
    ctx.fillRect(o.x, GROUND_Y - o.h, o.w, o.h);
  });

  const frame = Math.floor(elapsed / 90) % FRAME_COUNT;
  if (sprite.complete && sprite.naturalWidth) {
    ctx.drawImage(sprite, frame * FRAME_W, 0, FRAME_W, FRAME_H, PLAYER_X, player.y, PLAYER_W, PLAYER_H);
  } else {
    ctx.fillStyle = '#f5b301';
    ctx.fillRect(PLAYER_X, player.y, PLAYER_W, PLAYER_H);
  }
}

/** 닫기는 X 버튼으로만 — 배경 클릭·Esc 로는 닫히지 않는다. */
function onKeydown(e) {
  // 이름을 입력하는 중에는 스페이스가 점프가 아니라 글자여야 한다.
  const t = e.target;
  if (t instanceof HTMLElement && (t.tagName === 'INPUT' || t.tagName === 'TEXTAREA' || t.isContentEditable)) return;
  if (e.key === ' ' || e.key === 'ArrowUp' || e.key === 'Spacebar') {
    e.preventDefault();
    jump();
  }
}

onMounted(() => {
  ctx = canvasRef.value.getContext('2d');
  draw();
  window.addEventListener('keydown', onKeydown);
  loadBoard();
});

onBeforeUnmount(() => {
  cancelAnimationFrame(raf);
  window.removeEventListener('keydown', onKeydown);
});
</script>

<template>
  <div class="geumgaengrun" data-buddy-skip role="dialog" aria-modal="true" aria-label="금갱런 미니게임">
    <div class="geumgaengrun__dim" />
    <div class="geumgaengrun__card">
      <button type="button" class="geumgaengrun__close" aria-label="닫기" @click="emit('close')">✕</button>

      <div class="geumgaengrun__score">
        <span>점수 {{ score }}</span>
        <span class="geumgaengrun__best">최고 {{ highScore }}</span>
      </div>

      <canvas
        ref="canvasRef"
        width="600"
        height="200"
        class="geumgaengrun__canvas"
        @click="jump"
        @touchstart.prevent="jump"
      />

      <p v-if="status === 'ready'" class="geumgaengrun__hint">
        스페이스바 · 위 화살표 · 클릭/탭으로 점프! 공중에서 한 번 더 누르면 2단 점프.
      </p>
      <p v-else-if="status === 'over'" class="geumgaengrun__hint">
        게임 종료! 다시 눌러서 재도전하세요.
      </p>

      <!-- 기록 등록 — 게임이 끝났고 아직 안 올렸을 때만 -->
      <form v-if="status === 'over' && savedScore === null" class="grun-save" @submit.prevent="submitScore">
        <label class="grun-save__label" for="grun-name">이름을 남기고 명예의 전당에 도전!</label>
        <div class="grun-save__row">
          <input
            id="grun-name"
            v-model="playerName"
            class="grun-save__input"
            type="text"
            :maxlength="NAME_MAX"
            placeholder="이름 (최대 12자)"
            autocomplete="off"
          />
          <button type="submit" class="grun-save__btn" :disabled="saving">
            {{ saving ? '등록 중…' : `${score}점 등록` }}
          </button>
        </div>
        <p v-if="saveError" class="grun-save__error">{{ saveError }}</p>
      </form>

      <p v-else-if="savedScore !== null" class="grun-save__done">
        <template v-if="improved">🎉 <b>{{ savedScore }}점</b> 등록 완료! 지금 <b>{{ myRank }}위</b>예요.</template>
        <template v-else>{{ savedScore }}점! 내 최고 기록 <b>{{ myBest }}점</b>({{ myRank }}위)은 아직 그대로예요.</template>
      </p>

      <!-- 명예의 전당 -->
      <section class="grun-hall">
        <h3 class="grun-hall__title">명예의 전당</h3>

        <p v-if="boardError" class="grun-hall__empty">{{ boardError }}</p>
        <p v-else-if="!board.length" class="grun-hall__empty">
          아직 아무 기록도 없어요. 첫 주인공이 되어 주세요!
        </p>

        <template v-else>
          <ol class="grun-podium">
            <li
              v-for="(row, i) in podium"
              :key="`${row.name}-${row.score}-${i}`"
              class="grun-podium__slot"
              :class="`is-rank${i + 1}`"
            >
              <span class="grun-champ" aria-hidden="true">
                <i class="grun-champ__spark grun-champ__spark--a" />
                <i class="grun-champ__spark grun-champ__spark--b" />
                <i class="grun-champ__spark grun-champ__spark--c" />
                <span class="grun-champ__sprite" />
                <span class="grun-champ__medal">{{ i + 1 }}</span>
              </span>
              <span class="grun-podium__name">{{ row.name }}</span>
              <span class="grun-podium__score">{{ row.score }}</span>
              <span class="grun-podium__block">{{ i + 1 }}</span>
            </li>
          </ol>

          <ol v-if="runners.length" class="grun-rest">
            <li v-for="(row, i) in runners" :key="`${row.name}-${row.score}-${i}`">
              <span class="grun-rest__rank">{{ i + 4 }}</span>
              <span class="grun-rest__name">{{ row.name }}</span>
              <b class="grun-rest__score">{{ row.score }}</b>
            </li>
          </ol>
        </template>
      </section>
    </div>
  </div>
</template>
