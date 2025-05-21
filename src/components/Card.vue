<template>
  <div class="card" :class="[suitClass, { back: faceDown }]">
    <div v-if="!faceDown">
      <span class="rank">{{ rankDisplay }}</span>
      <span class="suit">{{ suitSymbol }}</span>
    </div>
  </div>
</template>

<script setup lang="ts">
import { computed, defineProps } from "vue";

type Suit = "S" | "H" | "D" | "C";
interface CardType {
  suit: Suit;
  rank: number;
}

// `faceDown` が true のときに裏面を描画
const props = defineProps<{
  card: CardType;
  faceDown?: boolean;
}>();

const suitSymbol = computed(() => {
  return { S: "♠", H: "♥", D: "♦", C: "♣" }[props.card.suit];
});
const rankDisplay = computed(() => {
  const r = props.card.rank;
  if (r === 1) return "A";
  if (r === 11) return "J";
  if (r === 12) return "Q";
  if (r === 13) return "K";
  return r.toString();
});
const suitClass = computed(() =>
  props.card.suit === "H" || props.card.suit === "D" ? "red" : "black"
);
</script>

<style scoped>
.card {
  width: 60px;
  height: 90px;
  border: 1px solid #333;
  border-radius: 6px;
  background: white;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  padding: 4px;
  box-sizing: border-box;
}
.card.red {
  color: red;
}
.card.black {
  color: black;
}
.rank {
  font-size: 1.2em;
}
.suit {
  font-size: 1.4em;
  line-height: 0.8;
}

/* 裏面用の強い色彩パターン */
.card.back {
  border: 1px solid #003300;
  background-color: #005500;
  background-image: repeating-linear-gradient(
    45deg,
    #006600 0,
    #006600 8px,
    #004400 8px,
    #004400 16px
  );
}
</style>
