<script setup>
import { computed, nextTick, onBeforeUnmount, onMounted, ref } from "vue";
import { words } from "./data.js";

const STORAGE_KEY = "swipe:v1";
const SWIPE_THRESHOLD = 135;

const queue = ref([]);
const learnedIds = ref([]);
const isLibraryOpen = ref(false);
const isDragging = ref(false);
const isAnimating = ref(false);
const dragX = ref(0);
const dragY = ref(0);
const startX = ref(0);
const startY = ref(0);
const activePointerId = ref(null);
const cardEl = ref(null);

const total = words.length;
const wordsById = new Map(words.map((item) => [item.id, item]));

function loadState() {
  try {
    const saved = JSON.parse(localStorage.getItem(STORAGE_KEY));
    const validIds = new Set(words.map((item) => item.id));
    const learned = Array.isArray(saved?.learnedIds)
      ? saved.learnedIds.filter((id) => validIds.has(id))
      : [];
    const learnedSet = new Set(learned);
    const savedQueue = Array.isArray(saved?.queue)
      ? saved.queue.filter((id) => validIds.has(id) && !learnedSet.has(id))
      : [];
    const queueSet = new Set(savedQueue);
    const missing = words
      .map((item) => item.id)
      .filter((id) => !learnedSet.has(id) && !queueSet.has(id));

    learnedIds.value = learned;
    queue.value = [...savedQueue, ...missing];
  } catch {
    queue.value = words.map((item) => item.id);
    learnedIds.value = [];
  }

  if (!queue.value.length && learnedIds.value.length !== total) {
    queue.value = words
      .map((item) => item.id)
      .filter((id) => !learnedIds.value.includes(id));
  }
}

function persistState() {
  localStorage.setItem(
    STORAGE_KEY,
    JSON.stringify({
      queue: queue.value,
      learnedIds: learnedIds.value,
    }),
  );
}

const currentCard = computed(() => wordsById.get(queue.value[0]) ?? null);
const visibleCards = computed(() =>
  queue.value
    .slice(0, 3)
    .map((id) => wordsById.get(id))
    .filter(Boolean),
);
const learnedWords = computed(() =>
  learnedIds.value
    .map((id) => wordsById.get(id))
    .filter(Boolean)
    .reverse(),
);
const currentPosition = computed(() => {
  if (!currentCard.value) return total;
  return Math.min(learnedIds.value.length + 1, total);
});
const dragProgress = computed(() =>
  Math.min(Math.abs(dragX.value) / SWIPE_THRESHOLD, 1),
);
const dragDirection = computed(() =>
  dragX.value < 0 ? "left" : dragX.value > 0 ? "right" : null,
);
const frontTransform = computed(() => {
  const rotation = dragX.value / 28;
  return `translate3d(${dragX.value}px, ${dragY.value}px, 0) rotate(${rotation}deg)`;
});

function resetDrag() {
  dragX.value = 0;
  dragY.value = 0;
  isDragging.value = false;
  activePointerId.value = null;
}

function setCardElement(el, index) {
  if (index === 0) cardEl.value = el;
}

function onPointerDown(event) {
  if (isAnimating.value || !currentCard.value || event.button > 0) return;
  isDragging.value = true;
  activePointerId.value = event.pointerId;
  startX.value = event.clientX;
  startY.value = event.clientY;
  cardEl.value?.setPointerCapture?.(event.pointerId);
}

function onPointerMove(event) {
  if (!isDragging.value || event.pointerId !== activePointerId.value) return;
  dragX.value = event.clientX - startX.value;
  dragY.value = (event.clientY - startY.value) * 0.18;
}

function onPointerUp(event) {
  if (!isDragging.value || event.pointerId !== activePointerId.value) return;
  cardEl.value?.releasePointerCapture?.(event.pointerId);
  const shouldSwipe = Math.abs(dragX.value) >= SWIPE_THRESHOLD;

  if (shouldSwipe) {
    completeSwipe(dragX.value < 0 ? "left" : "right");
  } else {
    resetDrag();
  }
}

function onPointerCancel() {
  if (!isDragging.value) return;
  resetDrag();
}

async function completeSwipe(direction) {
  if (isAnimating.value || !currentCard.value) return;
  isAnimating.value = true;
  isDragging.value = false;
  dragX.value =
    direction === "left" ? -window.innerWidth * 0.75 : window.innerWidth * 0.75;
  dragY.value = -18;

  await new Promise((resolve) => setTimeout(resolve, 260));

  const [current, ...rest] = queue.value;
  if (direction === "left") {
    if (!learnedIds.value.includes(current)) learnedIds.value.push(current);
    queue.value = rest;
  } else {
    // Возвращаем слово в хвост очереди, чтобы оно встретилось снова позже.
    queue.value = [...rest, current];
  }

  persistState();
  dragX.value = 0;
  dragY.value = 0;
  activePointerId.value = null;
  await nextTick();
  isAnimating.value = false;
}

function restoreWord(id) {
  learnedIds.value = learnedIds.value.filter((item) => item !== id);
  if (!queue.value.includes(id)) queue.value.push(id);
  persistState();
}

function resetProgress() {
  learnedIds.value = [];
  queue.value = words.map((item) => item.id);
  persistState();
  isLibraryOpen.value = false;
}

function speak(word) {
  if (!("speechSynthesis" in window)) return;
  window.speechSynthesis.cancel();
  const utterance = new SpeechSynthesisUtterance(word);
  utterance.lang = "en-US";
  utterance.rate = 0.86;
  window.speechSynthesis.speak(utterance);
}

function onKeydown(event) {
  if (isLibraryOpen.value || isAnimating.value) return;
  if (event.key === "ArrowLeft") completeSwipe("left");
  if (event.key === "ArrowRight") completeSwipe("right");
}

onMounted(() => {
  loadState();
  window.addEventListener("keydown", onKeydown);
});

onBeforeUnmount(() => {
  window.removeEventListener("keydown", onKeydown);
});
</script>

<template>
  <main class="app-shell">
    <header class="topbar">
      <div class="topbar-spacer" aria-hidden="true"></div>
      <a class="brand" href="#" aria-label="Swipe">Swipe</a>
      <div class="topbar-actions">
        <button
          class="learned-button"
          type="button"
          @click="isLibraryOpen = true"
        >
          Изученные слова
          <span v-if="learnedIds.length" class="learned-count">{{
            learnedIds.length
          }}</span>
        </button>
        <span class="progress-count">{{ currentPosition }} / {{ total }}</span>
      </div>
    </header>

    <section
      v-if="currentCard"
      class="study-area"
      aria-label="Карточки для изучения"
    >
      <div
        class="swipe-hint swipe-hint-left"
        :class="{ active: dragDirection === 'left' }"
        :style="{
          '--hint-progress': dragDirection === 'left' ? dragProgress : 0,
        }"
        aria-hidden="true"
      >
        <span class="hint-arrow">←</span>
        <span>изучено</span>
      </div>

      <div class="card-stage">
        <article
          v-for="(card, index) in visibleCards"
          :key="card.id"
          class="word-card"
          :class="[
            `word-card-${index}`,
            {
              'is-front': index === 0,
              'is-dragging': index === 0 && isDragging,
              'is-animating': index === 0 && isAnimating,
            },
          ]"
          :style="index === 0 ? { transform: frontTransform } : undefined"
          :ref="(el) => setCardElement(el, index)"
          @pointerdown="index === 0 && onPointerDown($event)"
          @pointermove="index === 0 && onPointerMove($event)"
          @pointerup="index === 0 && onPointerUp($event)"
          @pointercancel="index === 0 && onPointerCancel()"
        >
          <template v-if="index === 0">
            <button
              class="audio-button"
              type="button"
              aria-label="Прослушать произношение"
              @pointerdown.stop
              @click.stop="speak(card.word)"
            >
              <svg viewBox="0 0 24 24" aria-hidden="true">
                <path
                  d="M4 9.4v5.2h3.5l4.3 3.6V5.8L7.5 9.4H4Z"
                  fill="currentColor"
                />
                <path
                  d="M15 8.4c1.8 1.9 1.8 5.3 0 7.2M17.8 5.8c3.3 3.3 3.3 9.1 0 12.4"
                  fill="none"
                  stroke="currentColor"
                  stroke-width="1.6"
                  stroke-linecap="round"
                />
              </svg>
            </button>

            <div class="word-heading">
              <h1>{{ card.word }}</h1>
              <p class="transcription">{{ card.transcription }}</p>
              <p class="part">{{ card.part }}</p>
            </div>

            <div class="card-divider"></div>

            <div class="word-copy">
              <p class="example">{{ card.example }}</p>
              <p class="translation">{{ card.translation }}</p>
            </div>
          </template>
        </article>
      </div>

      <div
        class="swipe-hint swipe-hint-right"
        :class="{ active: dragDirection === 'right' }"
        :style="{
          '--hint-progress': dragDirection === 'right' ? dragProgress : 0,
        }"
        aria-hidden="true"
      >
        <span class="hint-arrow">→</span>
        <span>повторить</span>
      </div>
    </section>

    <section v-else class="finished-state">
      <p class="finished-kicker">Готово</p>
      <h1>Все слова изучены.</h1>
      <p>Ты можешь вернуть любое слово из списка изученных.</p>
      <button
        type="button"
        class="learned-button finished-button"
        @click="isLibraryOpen = true"
      >
        Изученные слова
      </button>
    </section>

    <Transition name="modal-fade">
      <div
        v-if="isLibraryOpen"
        class="modal-backdrop"
        @click.self="isLibraryOpen = false"
      >
        <section
          class="learned-panel"
          role="dialog"
          aria-modal="true"
          aria-labelledby="learned-title"
        >
          <div class="panel-header">
            <div>
              <p class="panel-kicker">{{ learnedIds.length }} из {{ total }}</p>
              <h2 id="learned-title">Изученные слова</h2>
            </div>
            <button
              class="close-button"
              type="button"
              aria-label="Закрыть"
              @click="isLibraryOpen = false"
            >
              ×
            </button>
          </div>

          <div v-if="learnedWords.length" class="learned-list">
            <article
              v-for="item in learnedWords"
              :key="item.id"
              class="learned-item"
            >
              <div>
                <h3>{{ item.word }}</h3>
                <p>{{ item.translation }}</p>
              </div>
              <button type="button" @click="restoreWord(item.id)">
                Вернуть
              </button>
            </article>
          </div>

          <div v-else class="empty-learned">
            Здесь появятся слова, которые ты отправишь свайпом влево.
          </div>

          <button
            v-if="learnedIds.length"
            class="reset-button"
            type="button"
            @click="resetProgress"
          >
            Сбросить прогресс
          </button>
        </section>
      </div>
    </Transition>
  </main>
</template>
