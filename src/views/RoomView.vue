<script setup lang="ts">
import {
  errMsgKey,
  PieceType,
  PlayerType,
  WinType,
  type BoardInfo,
  type RoomDetail,
} from "@/Type";
import { inject, ref, type Ref, onUnmounted } from "vue";
import { conn } from "@/logic/Connection";
import router from "@/router";

onUnmounted(() => {
  conn.off("gotRoom");
  conn.off("gotBoard");
});

const props = defineProps<{
  id: string;
}>();
const errMsg = inject<Ref<string | undefined>>(errMsgKey)!;
const playerType = ref(PlayerType.None);

conn.invoke("joinRoom", props.id).then((t: PlayerType) => {
  playerType.value = t;
  if (t == PlayerType.None) {
    router.push("/hall");
    errMsg.value = "The room doesn't exist.";
  }
});

const roomDetail = ref<RoomDetail>();
const startDisabled = ref(false);
const turn = ref(PlayerType.None);
const startGame = async () => {
  startDisabled.value = true;
  await conn.invoke("startGame");
  startDisabled.value = false;
};
conn.on("gotRoom", (detail: RoomDetail) => {
  roomDetail.value = detail;
});

const board = ref<PieceType[]>();
const result = ref(WinType.Null);
const go = (index: number) => {
  conn.invoke("go", index);
};
conn.on("gotBoard", (b: BoardInfo) => {
  board.value = b.board;
  turn.value = b.turn;
  result.value = b.result;
});
</script>

<template>
  <div class="grid px-6 grid-rows-4 grid-cols-4">
    <div
      class="col-start-1 col-span-1 text-gray-800 text-lg shadow-md shadow-slate-200"
    >
      Black: {{ roomDetail?.host }}
    </div>
    <div v-if="roomDetail?.ready || roomDetail?.started" class="col-span-1">
      {{ roomDetail?.started ? "Game" : roomDetail?.ready ? "Ready" : "" }}
    </div>
    <div
      class="col-start-3 col-span-1 text-gray-200 text-lg shadow-md shadow-slate-800"
    >
      White: {{ roomDetail?.guest }}
    </div>
    <button
      v-if="!(roomDetail?.started || playerType == PlayerType.Observer)"
      class="col-span-1 text-center p-1 bg-pink-300 hover:bg-pink-600 transition-colors"
      @click="startGame"
      :disabled="startDisabled"
    >
      {{
        playerType == PlayerType.Host
          ? "Start"
          : roomDetail?.ready
          ? "Unready"
          : "Ready"
      }}
    </button>
    <div
      class="row-span-3 col-span-3 grid grid-cols-5 grid-rows-5 divide-x divide-y divide-gray-400"
    >
      <button
        v-for="(piece, index) in board"
        class="text-center p-1 text-base cursor-pointer"
        :disabled="
          !roomDetail?.started || playerType != turn || piece != PieceType.Null
        "
        @click="go(index)"
      >
        {{
          piece == PieceType.Black ? "⚫" : piece == PieceType.White ? "⚪" : ""
        }}
      </button>
    </div>
    <div
      class="row-span-3 text-center text-lg font-bold p-5"
      v-if="roomDetail?.started"
    >
      Now<br />{{
        turn == PlayerType.Host ? "⚫" : turn == PlayerType.Guest ? "⚪" : ""
      }}
    </div>
    <div
      class="row-span-3 text-center text-lg font-bold p-5"
      v-else-if="result != WinType.Null"
    >
      {{ result == WinType.Tie ? "Tie" : "Win" }}
      <br />
      {{ result == WinType.Black ? "⚫" : result == WinType.White ? "⚪" : "" }}
    </div>
  </div>
</template>