<template>
  <div class="card-pile" @click="onClickPile">
    <!-- 何もない山なら空スロットを表示 -->
    <div v-if="cards.length === 0" class="empty-slot"></div>

    <!-- カードを重ねて表示 -->
    <div
      v-for="(card, idx) in cards"
      :key="card.id"
      class="card"
      :class="{ 'face-down': !card.faceUp }"
      :style="{ top: idx * overlap + 'px' }"
      @click.stop="onClickCard(idx)"
    >
      <!-- 表向きなら中身を表示 -->
      <template v-if="card.faceUp">
        <span class="rank">{{ rankName(card.rank) }}</span>
        <span class="suit">{{ suitSymbol(card.suit) }}</span>
      </template>
    </div>
  </div>
</template>

<script setup lang="ts">
import { defineProps, defineEmits } from "vue";

type Suit = "spades" | "hearts" | "diamonds" | "clubs";
interface CardType {
  id: string;
  suit: Suit;
  rank: number; // 1 = A, 11 = J, 12 = Q, 13 = K
  faceUp: boolean;
}

const props = defineProps<{
  /** この山に積まれたカード配列 */
  cards: CardType[];
  /** カード同士の重なり幅 (px) */
  overlap: number;
}>();

const emit = defineEmits<{
  /** 山全体をクリックしたとき */
  (e: "pile-click"): void;
  /** idx 番目のカードをクリックしたとき (0 が一番下) */
  (e: "card-click", idx: number): void;
}>();

/** 山全体をクリックしたとき */
function onClickPile() {
  emit("pile-click");
}

/** 個々のカードをクリックしたとき */
function onClickCard(idx: number) {
  emit("card-click", idx);
}

/** ランクを文字列に変換 */
function rankName(r: number) {
  if (r === 1) return "A";
  if (r === 11) return "J";
  if (r === 12) return "Q";
  if (r === 13) return "K";
  return String(r);
}

/** スートをシンボルに変換 */
function suitSymbol(s: Suit) {
  switch (s) {
    case "spades":
      return "♠";
    case "hearts":
      return "♥";
    case "diamonds":
      return "♦";
    case "clubs":
      return "♣";
  }
}
</script>

<style scoped>
.card-pile {
  position: relative;
  width: 80px;
  height: 120px;
}

/* 空のスロット */
.empty-slot {
  width: 100%;
  height: 100%;
  border: 2px dashed #ccc;
  border-radius: 6px;
}

/* カード本体 */
.card {
  position: absolute;
  width: 80px;
  height: 120px;
  border: 1px solid #333;
  border-radius: 6px;
  background: white;
  box-shadow: 1px 1px 3px rgba(0, 0, 0, 0.3);
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  padding: 4px;
  cursor: pointer;
  user-select: none;
}

/* 裏向きカード */
.face-down {
  background: #2a2a2a;
}

/* ランク、スートの見た目調整 */
.rank {
  font-size: 1.2em;
  font-weight: bold;
}
.suit {
  font-size: 1.4em;
  line-height: 1;
}
</style>
