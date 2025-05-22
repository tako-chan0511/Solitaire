<template>
  <div class="solitaire">
    <!-- ゲームコントロール -->
    <div class="game-controls">
      <button @click="initGame">再ゲーム</button>
      <button @click="undo" :disabled="historyStack.length <= 1">
        一つ戻す
      </button>
      <!-- 自動完了ボタン：条件付きで有効化 -->
      <button @click="autoMoveToFoundation" :disabled="!canAutoComplete">
        自動完了
      </button>
    </div>

    <!-- 上部：ストック／ワースト／ファンデーション -->
    <div class="top-row">
      <!-- ストック -->
      <div class="stock" @click="drawFromStock">
        <Card
          v-if="stock.length"
          :card="stock[stock.length - 1]"
          :face-down="true"
        />
        <div v-else class="empty-pile">空</div>
      </div>

      <!-- ワースト：重ねて表示 -->
      <div class="waste" @click="onWasteClick">
        <template v-if="waste.length">
          <div v-for="(c, i) in waste" :key="i" class="waste-card">
            <Card :card="c" :face-down="false" />
          </div>
        </template>
        <div v-else class="empty-pile">－</div>
      </div>

      <!-- ファンデーション４つ -->
      <div class="foundations">
        <div
          v-for="(pile, i) in foundations"
          :key="i"
          class="foundation"
          @click.stop="selectFoundation(i)"
          :class="{ 'selected-foundation': selectedFoundation === i }"
        >
          <Card
            v-if="pile.length"
            :card="pile[pile.length - 1]"
            :face-down="false"
          />
          <div v-else class="empty-pile">♢</div>
        </div>
      </div>
    </div>

    <!-- 下部：テーブル（７列） -->
    <div class="tableau">
      <div
        v-for="(col, ci) in tableaus"
        :key="ci"
        class="tableau-col"
        @click="onTableauClick(ci, 0)"
        :class="{
          highlight:
            selectedFoundation !== null &&
            foundations[selectedFoundation].length > 0 &&
            canMoveToTab(
              col,
              foundations[selectedFoundation][
                foundations[selectedFoundation].length - 1
              ]
            ),
        }"
      >
        <div
          v-for="(card, ri) in col"
          :key="ri"
          class="tableau-card"
          :class="{ 'face-up': card.faceUp, 'face-down': !card.faceUp }"
          @click.stop="onTableauClick(ci, ri)"
        >
          <Card :card="card" :face-down="!card.faceUp" />
        </div>
        <div v-if="!col.length" class="empty-pile">□</div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, watch, computed } from "vue";
import Card from "./Card.vue";

type Suit = "S" | "H" | "D" | "C";
interface CardType {
  suit: Suit;
  rank: number;
  faceUp: boolean;
}
// 履歴スナップショット型
interface Snapshot {
  stock: CardType[];
  waste: CardType[];
  foundations: CardType[][];
  tableaus: CardType[][];
}

// --- 1. 完了カード置き場選択用 ---
const selectedFoundation = ref<number | null>(null);
function selectFoundation(i: number) {
  if (foundations.value[i].length > 0) {
    selectedFoundation.value = i;
  }
}

// --- デッキ生成／シャッフル ---
function createDeck(): CardType[] {
  const suits: Suit[] = ["S", "H", "D", "C"];
  const deck: CardType[] = [];
  for (const s of suits) {
    for (let r = 1; r <= 13; r++) {
      deck.push({ suit: s, rank: r, faceUp: false });
    }
  }
  return deck;
}
function shuffle(deck: CardType[]) {
  for (let i = deck.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [deck[i], deck[j]] = [deck[j], deck[i]];
  }
}

// --- リアクティブな状態 ---
const stock = ref<CardType[]>([]);
const waste = ref<CardType[]>([]);
const foundations = ref<CardType[][]>([[], [], [], []]);
const tableaus = ref<CardType[][]>([[], [], [], [], [], [], []]);

// --- 履歴スタック & 操作前履歴保存 ---
const historyStack = ref<Snapshot[]>([]);
function saveHistory() {
  historyStack.value.push({
    stock: stock.value.map((c) => ({ ...c })),
    waste: waste.value.map((c) => ({ ...c })),
    foundations: foundations.value.map((col) => col.map((c) => ({ ...c }))),
    tableaus: tableaus.value.map((col) => col.map((c) => ({ ...c }))),
  });
}
function beforeAction() {
  saveHistory();
}
function undo() {
  if (historyStack.value.length <= 1) return;
  historyStack.value.pop();
  const prev = historyStack.value[historyStack.value.length - 1];
  stock.value = prev.stock.map((c) => ({ ...c }));
  waste.value = prev.waste.map((c) => ({ ...c }));
  foundations.value = prev.foundations.map((col) => col.map((c) => ({ ...c })));
  tableaus.value = prev.tableaus.map((col) => col.map((c) => ({ ...c })));
}

// --- ゲーム初期化 ---
function initGame() {
  const deck = createDeck();
  shuffle(deck);
  tableaus.value = [[], [], [], [], [], [], []];
  for (let i = 0; i < 7; i++) {
    for (let j = 0; j <= i; j++) {
      const c = deck.pop()!;
      c.faceUp = j === i;
      tableaus.value[i].push(c);
    }
  }
  stock.value = deck;
  waste.value = [];
  foundations.value = [[], [], [], []];
  historyStack.value = [];
  saveHistory();
}
initGame();

// --- 移動可能判定 ---
function canMoveToFound(pile: CardType[], c: CardType) {
  if (!pile.length) return c.rank === 1;
  const top = pile[pile.length - 1];
  return top.suit === c.suit && top.rank + 1 === c.rank;
}
function canMoveToTab(pile: CardType[], c: CardType) {
  if (!pile.length) return c.rank === 13;
  const top = pile[pile.length - 1];
  const isRed = (s: Suit) => s === "H" || s === "D";
  return isRed(top.suit) !== isRed(c.suit) && top.rank === c.rank + 1;
}

// --- 自動完了条件 ---
const canAutoComplete = computed(() => {
  const noStock = stock.value.length === 0;
  const allTableauFaceUp = tableaus.value.every((col) =>
    col.every((card) => card.faceUp)
  );
  return noStock && allTableauFaceUp;
});

// --- 自動完了処理 ---
function autoMoveToFoundation() {
  let moved: boolean;
  do {
    moved = false;
    // (1) waste → foundation
    if (waste.value.length) {
      const c = waste.value[waste.value.length - 1];
      for (let fi = 0; fi < 4; fi++) {
        if (canMoveToFound(foundations.value[fi], c)) {
          beforeAction();
          waste.value.pop();
          foundations.value[fi].push(c);
          moved = true;
          break;
        }
      }
    }
    if (moved) continue;

    // (2) tableau → foundation
    for (let ci = 0; ci < tableaus.value.length; ci++) {
      const col = tableaus.value[ci];
      if (!col.length) continue;
      const c = col[col.length - 1];
      if (!c.faceUp) continue;
      for (let fi = 0; fi < 4; fi++) {
        if (canMoveToFound(foundations.value[fi], c)) {
          beforeAction();
          col.pop();
          foundations.value[fi].push(c);
          moved = true;
          break;
        }
      }
      if (moved) break;
    }
  } while (moved);
}

// --- ストックめくり＆自動リサイクル ---
function drawFromStock() {
  beforeAction();
  if (stock.value.length) {
    const c = stock.value.pop()!;
    c.faceUp = true;
    waste.value.push(c);
  } else {
    while (waste.value.length) {
      const c = waste.value.pop()!;
      c.faceUp = false;
      stock.value.push(c);
    }
  }
}
watch(stock, (newStock) => {
  if (newStock.length === 0 && waste.value.length > 0) {
    while (waste.value.length) {
      const c = waste.value.pop()!;
      c.faceUp = false;
      stock.value.push(c);
    }
  }
});

// --- ワースト→ファンデーション 移動 ---
function moveWasteToFoundation(fi: number) {
  if (!waste.value.length) return;
  beforeAction();
  const c = waste.value[waste.value.length - 1];
  if (canMoveToFound(foundations.value[fi], c)) {
    waste.value.pop();
    foundations.value[fi].push(c);
  }
}
function onWasteClick() {
  if (!waste.value.length) return;
  beforeAction();
  const c = waste.value[waste.value.length - 1];
  // foundation 優先
  for (let i = 0; i < 4; i++) {
    if (canMoveToFound(foundations.value[i], c)) {
      moveWasteToFoundation(i);
      return;
    }
  }
  // tableau
  for (let i = 0; i < 7; i++) {
    if (canMoveToTab(tableaus.value[i], c)) {
      waste.value.pop();
      tableaus.value[i].push(c);
      return;
    }
  }
}

// --- 場札クリック／完了カード置き場→場札移動対応 ---
function onTableauClick(ci: number, ri: number) {
  // const col1 = tableaus.value[ci];
  // // 空列なら何もしない
  // if (!col1.length) return;

  // const seq1 = col.slice(ri);
  // // 切り出したシーケンスが空、または先頭カードが表向きでなければ何もしない
  // if (!seq1.length || !seq1[0].faceUp) return;
  // (1) 完了カード置き場から戻す
  if (selectedFoundation.value !== null) {
    const fi = selectedFoundation.value;
    const pile = foundations.value[fi];
    const c = pile[pile.length - 1];
    if (canMoveToTab(tableaus.value[ci], c)) {
      beforeAction();
      pile.pop();
      tableaus.value[ci].push(c);
    }
    selectedFoundation.value = null;
    return;
  }

  // (2) 通常移動
  const col = tableaus.value[ci];
  const seq = col.slice(ri);
  if (!seq[0].faceUp) return;
  beforeAction();

  // foundation へ１枚
  if (seq.length === 1) {
    for (let i = 0; i < 4; i++) {
      if (canMoveToFound(foundations.value[i], seq[0])) {
        col.splice(ri);
        foundations.value[i].push(seq[0]);
        if (col.length && !col[col.length - 1].faceUp) {
          col[col.length - 1].faceUp = true;
        }
        return;
      }
    }
  }
  // 他テーブルへ
  for (let ti = 0; ti < 7; ti++) {
    if (ti === ci) continue;
    if (canMoveToTab(tableaus.value[ti], seq[0])) {
      col.splice(ri);
      tableaus.value[ti].push(...seq);
      if (col.length && !col[col.length - 1].faceUp) {
        col[col.length - 1].faceUp = true;
      }
      return;
    }
  }
}
</script>

<style scoped>
.solitaire {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}
.game-controls {
  display: flex;
  gap: 0.5rem;
}
.top-row {
  display: flex;
  align-items: flex-start;
  gap: 1.5rem;
}
.stock,
.foundation {
  width: 60px;
  height: 90px;
  position: relative;
}
.foundations {
  display: flex;
  gap: 0.5rem;
}
.waste {
  position: relative;
  width: 60px;
  height: 90px;
}
.waste-card {
  position: absolute;
  top: 0;
  left: 0;
}
.tableau {
  display: flex;
  gap: 0.5rem;
  margin-top: 70px;
}
.tableau-col {
  width: 60px;
}
.tableau-card {
  margin-top: -70px;
  position: relative;
  z-index: 1;
}
.face-down {
  opacity: 0.2;
}
.empty-pile {
  width: 60px;
  height: 90px;
  border: 2px dashed #aaa;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #aaa;
}
.tableau-col > .empty-pile {
  margin-top: -70px;
  position: relative;
  z-index: 1;
}
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

/* 表示要素 */
.rank {
  font-size: 1.2em;
}
.suit {
  font-size: 1.4em;
  line-height: 0.8;
}

.selected-foundation {
  outline: 2px solid #f00;
}

.tableau-col.highlight {
  box-shadow: 0 0 8px 2px rgba(255, 255, 0, 0.8);
}
</style>
