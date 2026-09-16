<script setup>
import { computed, onMounted, onUnmounted, ref } from 'vue';
import {
  BEG_ROTATE_MS,
  GAME_CLICKS,
  GAME_CLICK_GAP,
  buddyState,
  nudge,
  openGeumgaengRun,
  pickBegMessage,
  pickIdleMessage,
  prefersReducedMotion,
} from '../stores/buddy';

defineProps({
  /** 상세 패널이 열린 넓은 화면에서는 패널을 피해 왼쪽으로 비켜선다. */
  shifted: { type: Boolean, default: false },
  /** 좁은 화면에서 상세·목록 시트가 덮고 있을 때는 화면 밖으로 잠시 내려간다. */
  tucked: { type: Boolean, default: false },
});

const typing = computed(() => buddyState.animating);

/**
 * 숨은 미니게임을 아무도 못 찾고 지나가서, 게임이 닫혀 있는 동안에는
 * 금갱이가 계속 "여기 5번 눌러달라"고 조른다. 게임이 열리면 조용해진다.
 */
const begging = computed(() => !buddyState.gameOpen);

/** 지금까지 연타한 횟수(0~4). 손을 멈추면 초기화된다. */
const streak = ref(0);
const begMessage = ref(pickBegMessage());

/** 말풍선은 순간 메시지(일하는 중·연타 반응)가 우선, 없으면 조르는 문구가 계속 떠 있다. */
const bubble = computed(() => buddyState.message || (begging.value ? begMessage.value : ''));
const beggingBubble = computed(() => !buddyState.message && begging.value);

const ctaText = computed(() => {
  const left = GAME_CLICKS - streak.value;
  if (streak.value === 0) return `여기 ${GAME_CLICKS}번 눌러주세요!`;
  return left === 1 ? '한 번만 더!' : `${left}번 더!`;
});

const spriteLabel = computed(() =>
  begging.value
    ? `금갱이 — 청약 길잡이 캐릭터. 연속으로 ${GAME_CLICKS}번 누르면 숨은 미니게임 금갱런이 열립니다.`
    : '금갱이 — 청약 길잡이 캐릭터'
);

/** 아무도 안 눌러도 혼자 일하고 있는 척 — 7~14초마다 잠깐씩 타자를 친다. */
const idleTimer = ref(null);
function scheduleIdleBurst() {
  const wait = 7000 + Math.random() * 7000;
  idleTimer.value = setTimeout(() => {
    if (!buddyState.animating && !document.hidden) nudge(1400, pickIdleMessage());
    scheduleIdleBurst();
  }, wait);
}

/** 같은 문장만 반복하면 잔소리처럼 들려서 주기적으로 갈아 끼운다. */
const begTimer = ref(null);
function scheduleBegRotate() {
  begTimer.value = setTimeout(() => {
    if (begging.value && !document.hidden) {
      let next = pickBegMessage();
      if (next === begMessage.value) next = pickBegMessage();
      begMessage.value = next;
    }
    scheduleBegRotate();
  }, BEG_ROTATE_MS);
}

/** 연타가 끊기면 진행 표시를 되돌린다. */
const streakTimer = ref(null);
let lastPoke = 0;

function poke() {
  const now = Date.now();
  if (now - lastPoke > GAME_CLICK_GAP) streak.value = 0;
  lastPoke = now;
  streak.value += 1;
  clearTimeout(streakTimer.value);

  if (streak.value >= GAME_CLICKS) {
    streak.value = 0;
    openGeumgaengRun();
    return;
  }

  streakTimer.value = setTimeout(() => {
    streak.value = 0;
  }, GAME_CLICK_GAP);

  const left = GAME_CLICKS - streak.value;
  nudge(1000, left === 1 ? '한 번만 더요!!' : `${left}번만 더 눌러주세요!`);
}

onMounted(() => {
  if (!prefersReducedMotion()) scheduleIdleBurst();
  scheduleBegRotate();
});
onUnmounted(() => {
  clearTimeout(idleTimer.value);
  clearTimeout(begTimer.value);
  clearTimeout(streakTimer.value);
});
</script>

<template>
  <div
    class="buddy"
    :class="{ 'is-shifted': shifted, 'is-tucked': tucked, 'is-begging': begging }"
    data-buddy-skip
  >
    <transition name="buddy-bubble">
      <p v-if="bubble" class="buddy__bubble" :class="{ 'is-beg': beggingBubble }">{{ bubble }}</p>
    </transition>

    <!-- 몇 번 남았는지까지 보여 주는 안내. 스프라이트와 같은 동작이라 눌러도 카운트된다. -->
    <button v-if="begging" type="button" class="buddy__cta" tabindex="-1" aria-hidden="true" @click="poke">
      <span>{{ ctaText }}</span>
      <span class="buddy__dots">
        <i v-for="n in GAME_CLICKS" :key="n" :class="{ 'is-on': n <= streak }" />
      </span>
    </button>

    <span class="buddy__stage">
      <template v-if="begging">
        <span class="buddy__halo" aria-hidden="true" />
        <span class="buddy__halo buddy__halo--late" aria-hidden="true" />
      </template>

      <button
        type="button"
        class="buddy__sprite"
        :class="{ 'is-typing': typing }"
        :aria-label="spriteLabel"
        @click="poke"
      />
    </span>
  </div>
</template>
