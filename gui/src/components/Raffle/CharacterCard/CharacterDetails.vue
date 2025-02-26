<template>
  <div>
    <CharacterOptions :show="showOptions" />

    <div class="flex justify-between">
      <h3 v-if="!Object.keys(winners).length" class="text-xl font-semibold">
        Toyhouse Raffle
      </h3>
      <div v-else>
        <h3 class="text-xl font-semibold">Congrats to the winners!</h3>
        <div class="flex items-center gap-3">
          <p class="text-sm">{{ new Date().toLocaleString() }}</p>
          <DownloadIcon class="w-4 h-4 cursor-pointer" @click="exportWinners" />
        </div>
      </div>
      <CogIcon
        class="z-50 transition duration-900 ease-out sticky"
        :class="{ rotate: showOptions }"
        @click="toggleShowoptions"
      />
    </div>

    <hr class="my-2" />

    <div class="items-center flex-wrap md:flex md:justify-between">
      <template v-if="Object.keys(winners).length === 0">
        <div class="flex flex-col items-center gap-2 md:flex-row">
          <img
            :src="mainCharacter.image"
            class="border border-toyhouse-border-primary p-2 bg-white rounded-md w-20 md:w-34 xl:w-44"
          />
          <div>
            <h3 class="font-light md:text-xl">{{ mainCharacter.name }}</h3>
            <div
              class="flex text-toyhouse-blue-primary transition duration-300 ease-out font-semibold hover:text-toyhouse-blue-secondary"
            >
              <UserIcon />
              <h3>
                <a
                  :href="mainCharacter.owner.profile"
                  target="_blank"
                  class="md:text-xl"
                  >{{ mainCharacter.owner.name }}</a
                >
              </h3>
            </div>
          </div>
        </div>
      </template>
      <template v-else>
        <div class="flex justify-center items-center flex-wrap">
          <template v-for="winner in winners" :key="winner.profile.id">
            <div class="flex-col p-1">
              <img
                :src="winner.profile.image"
                class="border border-toyhouse-border-primary p-1 bg-white rounded-md w-14 mx-auto md:w-20"
              />
              <h4 class="text-center" :title="winner.profile.name">
                {{ truncateName(winner.profile.name) }}
              </h4>
            </div>
          </template>
        </div>
      </template>

      <div class="flex-col">
        <div v-if="Object.keys(list).length" class="my-1 md:my-2">
          <h1 class="hidden font-semibold md:block">Winner count:</h1>
          <input
            v-model="winnerCount"
            class="p-1 border border-toyhouse-border-primary outline-0 rounded-md rounded-r-none w-full indent-1"
            placeholder="Winner count..."
          />
        </div>
        <div
          class="flex justify-center text-sm mt-1 md:m-0 lg:text-lg gap-1 md:justify-start"
        >
          <button
            class="bg-red-600 text-white p-2 rounded-md transition duration-300 ease-out hover:bg-red-700"
            type="button"
            @click="resetRaffle"
          >
            Reset raffle
          </button>
          <template v-if="Object.keys(list).length">
            <button
              class="bg-toyhouse-blue-primary text-white p-2 rounded-md transition duration-300 ease-out hover:bg-toyhouse-blue-secondary"
              type="button"
              @click="pick"
            >
              Pick Winners
            </button>
          </template>
          <template v-else>
            <button
              class="bg-toyhouse-blue-primary text-white p-2 rounded-md transition duration-300 ease-out hover:bg-toyhouse-blue-secondary"
              type="button"
              @click="loadParticipants"
            >
              Load participants
            </button>
          </template>
        </div>
      </div>
    </div>
  </div>
</template>
<script setup lang="ts">
import { useRaffleStore } from "../../../stores/raffleOptions.ts";
import { useMessagesStore } from "../../../stores/messagesStore.ts";
import { storeToRefs } from "pinia";
import { computed, ref } from "vue";
import CogIcon from "../../../assets/components/CogIcon.vue";
import UserIcon from "../../../assets/components/UserIcon.vue";
import CharacterOptions from "./CharacterOptions.vue";
import { useParticipantsStore } from "../../../stores/participantsStore.ts";
import { getRaffleTicketsForAll } from "../../../helpers/requests.ts";
import { confirm } from "@tauri-apps/api/dialog";
import DownloadIcon from "@/assets/components/DownloadIcon.vue";
import { invoke } from "@tauri-apps/api";
import { open } from "@tauri-apps/api/shell";

let optionsStore = useRaffleStore();
let pStore = useParticipantsStore();
let mStore = useMessagesStore();
let { setParticipants, setWinners, pickWinners, deleteParticipants } = pStore;
let { setLoading, setError, clearLoading, clearError } = mStore;
let { list, winners } = storeToRefs(pStore);
let { options } = storeToRefs(optionsStore);
let mainOpt = computed(() => options.value[0]);
let mainCharacter = computed(() => mainOpt.value.character);
let showOptions = ref(false);
const winnerCount = ref(null);

const toggleShowoptions = () => (showOptions.value = !showOptions.value);
const resetRaffle = async () => {
  let conf = await confirm("Are you sure you wish to reset all data?");
  if (!conf) return;

  options.value = [];
  deleteParticipants();
};

const loadParticipants = async () => {
  setLoading("Loading participants...");
  try {
    let json = await getRaffleTicketsForAll(options.value);
    setParticipants(json);
  } catch (err: any) {
    setError(err);
    setTimeout(() => clearError(), 1500);
  }
  clearLoading();
};

const pick = () => {
  console.log(winnerCount.value, Object.keys(list.value).length);
  if (winnerCount.value === null) {
    console.error("Winner Count is null");
    return;
  }

  if (winnerCount.value > Object.keys(list.value).length) {
    setError("Winner count cannot be higher than participant count!");
    setTimeout(() => clearError(), 1500);
    return;
  }
  const res = pickWinners(winnerCount.value);

  setWinners(res);
};

const truncateName = (name: string) => {
  if (name.length < 10) return name;

  return name.slice(0, 7) + "...";
};

const exportWinners = async () => {
  let winnersStr = "";

  winnersStr += `Raffle Winners (${new Date().toLocaleString()})\n\n`;
  Object.values(winners.value).forEach((winner) => {
    winnersStr += `${winner.profile.name}\n`;
  });

  const path: string = await invoke("download_raffle_results", {
    characterName: mainCharacter.value.name,
    results: winnersStr,
  });

  open(path, "open");
};
</script>
<style>
.rotate {
  transform: rotate(-90deg);
  transition: transform 2s ease-out;
}
</style>
