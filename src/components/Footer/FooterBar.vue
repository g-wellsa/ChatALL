<template>
  <div class="footer">
    <v-textarea
      v-model="prompt"
      auto-grow
      max-rows="8.5"
      rows="1"
      density="comfortable"
      hide-details
      variant="solo"
      :placeholder="$t('footer.promptPlaceholder')"
      autofocus
      @keydown="filterEnterKey"
      style="min-width: 390px"
    ></v-textarea>
    <v-btn
      color="primary"
      elevation="2"
      class="margin-bottom"
      :disabled="
        prompt.trim() === '' || Object.values(activeBots).every((bot) => !bot)
      "
      @click="sendPromptToBots"
    >
      {{ $t("footer.sendPrompt") }}
    </v-btn>
    <div class="bot-logos margin-bottom">
      <BotLogo
        v-for="(bot, index) in bots"
        :key="index"
        :bot="bot"
        :active="activeBots[bot.getClassname()]"
        @click="toggleSelected(bot)"
      ></BotLogo>
    </div>
    <MakeAvailableModal v-model:open="isMakeAvailableOpen" :bot="clickedBot" />
  </div>
</template>

<script setup>
import { ref, computed, onBeforeMount, reactive } from "vue";
import { useStore } from "vuex";

// Components
import MakeAvailableModal from "@/components/MakeAvailableModal.vue";
import BotLogo from "./BotLogo.vue";

// Composables
import { useMatomo } from "@/composables/matomo";

import _bots from "@/bots";

const { ipcRenderer } = window.require("electron");

const store = useStore();
const matomo = useMatomo();

const bots = ref(_bots.all);
const activeBots = reactive({});
const favBots = computed(() => {
  const _favBots = [];
  store.getters.currentChat.favBots.forEach((favBot) => {
    _favBots.push({
      ...favBot,
      instance: _bots.getBotByClassName(favBot.classname),
    });
  });
  return _favBots;
});

const prompt = ref("");
const clickedBot = ref(null);
const isMakeAvailableOpen = ref(false);

const setBotSelected = (param) => store.commit("SET_BOT_SELECTED", param);

function updateActiveBots() {
  for (const favBot of favBots.value) {
    activeBots[favBot.classname] =
      favBot.instance.isAvailable() && favBot.selected;
  }
}

// Send the prompt when the user presses enter and prevent the default behavior
// But if the shift, ctrl, alt, or meta keys are pressed, do as default
function filterEnterKey(event) {
  if (
    event.keyCode == 13 &&
    !event.shiftKey &&
    !event.ctrlKey &&
    !event.altKey &&
    !event.metaKey
  ) {
    event.preventDefault();
    sendPromptToBots();
  }
}

function sendPromptToBots() {
  if (prompt.value.trim() === "") return;
  if (Object.values(activeBots).every((bot) => !bot)) return;

  const toBots = bots.value.filter((bot) => activeBots[bot.getClassname()]);

  store.dispatch("sendPrompt", {
    prompt: prompt.value,
    bots: toBots,
  });

  // Clear the textarea after sending the prompt
  prompt.value = "";

  matomo.value?.trackEvent(
    "prompt",
    "send",
    "Active bots count",
    toBots.length,
  );
}

async function toggleSelected(bot) {
  const botClassname = bot.getClassname();
  let selected = false;
  if (activeBots[botClassname]) {
    selected = false;
  } else {
    selected = true;
    if (!bot.isAvailable()) {
      const availability = await bot.checkAvailability();
      if (!availability) {
        clickedBot.value = bot;
        // Open the bot's settings dialog
        isMakeAvailableOpen.value = true;
      }
    }
  }
  setBotSelected({ botClassname, selected });
  updateActiveBots();
}

onBeforeMount(async () => {
  favBots.value.forEach(async (favBot) => {
    await favBot.instance.checkAvailability();
    updateActiveBots();
  });

  // Listen message trigged by main process
  ipcRenderer.on("CHECK-AVAILABILITY", async (event, url) => {
    const botsToCheck = bots.value.filter((bot) => bot.getLoginUrl() === url);
    botsToCheck.forEach(async (bot) => {
      await bot.checkAvailability();
      updateActiveBots();
    });
  });
});
</script>

<style>
.footer {
  position: fixed;
  bottom: 0;
  left: 0;
  width: 100%;
  background-color: rgba(243, 243, 243, 0.7);
  display: flex;
  align-items: flex-end;
  justify-content: space-between;
  padding: 8px 16px;
  gap: 8px;
  box-sizing: border-box;
}

.bot-logos {
  display: flex;
  flex-wrap: wrap;
  gap: 4px;
}

.margin-bottom {
  margin-bottom: 5px;
}

/* Override default style of vuetify v-textarea */
.v-textarea--auto-grow textarea {
  overflow: auto !important;
}
</style>
