<template>
<v-app
  id="app"
  :style="cssVars"
>
  <div
    id="main-content"
  >

    <div
      id="iframes-container"
    >
      <iframe
        v-for="index in wwtCount"
        :key="index"
        :id="`wwt-${index-1}`"
        :name="`wwt-${index-1}`"
        :src="`https://web.wwtassets.org/research/latest/?origin=${origin}`"
        class="wwt-iframe"
        allow="accelerometer; clipboard-write; gyroscope"
        allowfullscreen
        frameborder="0"
      >
      </iframe>
    </div>

    <transition name="fade">
      <div
        class="modal"
        id="modal-loading"
        v-show="isLoading"
      >
        <div class="container">
          <div class="spinner"></div>
          <p>Loading …</p>
        </div>
      </div>
    </transition>


    <!-- This block contains the elements (e.g. icon buttons displayed at/near the top of the screen -->

    <div id="top-content">
      <div id="left-buttons">
        <v-btn
          :color="buttonColor"
          @click="move([0,1])"
        >
          Move both
        </v-btn>
        <v-btn
          :color="buttonColor"
          @click="move([0])"
        >
          Move left
        </v-btn>
        <v-btn
          :color="buttonColor"
          @click="move([1])"
        >
          Move right
        </v-btn>
      </div>
      <div id="center-buttons">
      </div>
      <div id="right-buttons">
      </div>
    </div>


    <!-- This block contains the elements (e.g. the project icons) displayed along the bottom of the screen -->

    <div id="bottom-content">
      <div id="body-logos" v-if= "!smallSize">
        <credit-logos/>
      </div>
    </div>


  </div>
</v-app>
</template>

<script setup lang="ts">
import { ref, computed, onMounted } from "vue";
import { useDisplay } from "vuetify";

export interface MultiDemoProps {
  wwtCount: number;
}

const origin = ref(window.location.origin);
const { smAndDown } = useDisplay();

function getWindow(index: number): Window | null {
  const idx = Math.round(index);
  return (window[`wwt-${idx}`] as Window) ?? null;
}

const props = withDefaults(defineProps<MultiDemoProps>(), {
  wwtCount: 2,
});

// eslint-disable-next-line @typescript-eslint/no-explicit-any
function sendMessage(index: number, message: Record<string, any>) {
  const frameWindow = getWindow(index);
  frameWindow?.postMessage(message, "https://web.wwtassets.org/");
}

// eslint-disable-next-line @typescript-eslint/no-explicit-any
function sendToAll(message: Record<string, any>) {
  for (let i = 0; i < props.wwtCount; i++) {
    sendMessage(i, message);
  }
}

function setup() {
  sendToAll({
    event: "modify_settings",
    settings: [["hideAllChrome", true]],
    target: "app",
  });

  sendMessage(0, {
    event: "setting_set",
    setting: "showAltAzGrid",
    value: true,
  });

  sendMessage(1, {
    event: "set_background_by_name",
    name: "Solar System",
  });
  sendMessage(1, {
    event: "set_foreground_by_name",
    name: "Solar System",
  });
}

function move(indices: number[]) {
  indices.forEach(index => {
    sendMessage(index, {
      event: "center_on_coordinates",
      ra: 0,
      dec: 0,
      fov: 30,
      instant: false,
    });
  });
}

const intervals: Map<number, string> = new Map();
const loaded = new Array(props.wwtCount).fill(false);
window.addEventListener("message", (message) => {
  const data = message.data;
  if (data.type === "wwt_ping_pong") {
    const idx = Number(data.sessionId);
    clearInterval(intervals[idx]);
    loaded[idx] = true;
  }
  const sourceWindow = message.source as Window;
  let index = -1;
  for (let i = 0; i < props.wwtCount; i++) {
    if (window[`wwt-${i}`] == sourceWindow) {
      index = i;
      break;
    }
  }
  if (index >= 0) {
    console.log(`Message received from iframe ${index}`);
    console.log(message);
  }
});

function setUpPings() {
  for (let i = 0; i < props.wwtCount; i++) {
    const strID = String(i);
    const interval = setInterval(() => {
      sendMessage(i, {
        type: "wwt_ping_pong",
        threadId: strID,
        sessionId: strID,
      });
    }, 100);
    intervals[i] = interval;
  }
}

const layersLoaded = ref(false);
const positionSet = ref(false);
const accentColor = ref("#ffffff");
const buttonColor = ref("#ffffff");

onMounted(() => {
  setUpPings();
  const interval = setInterval(() => {
    if (document.readyState === "complete" &&
        loaded.every(Boolean)) {
      setup();
      layersLoaded.value = true;
      positionSet.value = true;
      clearInterval(interval);
    }
  }, 100);
});

const ready = computed(() => layersLoaded.value && positionSet.value);

/* `isLoading` is a bit redundant here, but it could potentially have independent logic */
const isLoading = computed(() => !ready.value);

/* Properties related to device/screen characteristics */
const smallSize = computed(() => smAndDown.value);

/* This lets us inject component data into element CSS */
const cssVars = computed(() => {
  return {
    "--accent-color": accentColor.value,
    "--app-content-height": "100%",
  };
});
</script>

<style lang="less">
@font-face {
  font-family: "Highway Gothic Narrow";
  src: url("./assets/HighwayGothicNarrow.ttf");
}

:root {
  --default-font-size: clamp(0.7rem, min(1.7vh, 1.7vw), 1.1rem);
  --default-line-height: clamp(1rem, min(2.2vh, 2.2vw), 1.6rem);
}

html {
  height: 100%;
  margin: 0;
  padding: 0;
  background-color: #000;
  overflow: hidden;

  
  -ms-overflow-style: none;
  // scrollbar-width: none;
}

body {
  position: fixed;
  width: 100%;
  height: 100%;
  margin: 0;
  padding: 0;
  overflow: hidden;

  font-family: Verdana, Arial, Helvetica, sans-serif;
}

#app {
  height: 100%;
  width: 100%;
}

#main-content {
  position: fixed;
  width: 100%;
  height: var(--app-content-height);
  overflow: hidden;

  transition: height 0.1s ease-in-out;
}

#iframes-container {
  display: grid;
  grid-template-columns: 1fr 1fr;
  width: 100%;
  height: 100%;

  iframe {
    width: 100%;
    height: 100%;
  }
}

.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.3s;
}
.fade-enter,
.fade-leave-to {
  opacity: 0;
}

.modal {
  position: absolute;
  top: 0px;
  left: 0px;
  width: 100%;
  height: 100%;
  z-index: 100;
  color: #fff;
  background-color: rgba(0, 0, 0, 0.7);
  display: flex;
  align-items: center;
  justify-content: center;
}

#modal-loading {
  background-color: #000;
  .container {
    display: flex;
    flex-direction: row;
    align-items: center;
    justify-content: center;
    .spinner {
      background-image: url("https://projects.cosmicds.cfa.harvard.edu/cds-website/misc/lunar_loader.gif");
      background-repeat: no-repeat;
      background-size: contain;
      width: 3rem;
      height: 3rem;
    }
    p {
      margin: 0 0 0 1rem;
      padding: 0;
      font-size: 150%;
    }
  }
}

#top-content {
  position: absolute;
  top: 1rem;
  left: 1rem;
  width: calc(100% - 2rem);
  pointer-events: none;
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
}

#left-buttons {
  display: flex;
  flex-direction: column;
  gap: 10px;
  
  .v-btn {
    pointer-events: auto;
  }
}

#right-buttons {
  display: flex;
  flex-direction: column;
  gap: 10px;
  align-items: flex-end;
  height: auto;
}

#bottom-content {
  display: flex;
  flex-direction: column;
  position: absolute;
  bottom: 1rem;
  right: 1rem;
  width: calc(100% - 2rem);
  pointer-events: none;
  align-items: center;
  gap: 5px;
}

#splash-overlay {
  position: fixed;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
}

#splash-screen {
  color: #FFFFFF;
  background-color: #000000;
  display: flex;
  flex-direction: column;
  flex-wrap: wrap;
  align-content: center;
  justify-content: space-around;

  font-family: 'Highway Gothic Narrow', 'Roboto', sans-serif;
  font-size: min(8vw, 7vh);

  border-radius: 10%;
  border: min(1.2vw, 0.9vh) solid var(--accent-color);
  overflow: auto;
  padding-top: 4rem;
  padding-bottom: 1rem;

  @media (max-width: 699px) {
    max-height: 80vh;
    max-width: 90vw;
  }

  @media (min-width: 700px) {
    max-height: 85vh;
    max-width: min(70vw, 800px);
  }

  div {
    margin-inline: auto;
    text-align: center;
  }

  .small {
    font-size: var(--default-font-size);
    font-weight: bold;
  }

  #close-splash-button {
    position: absolute;
    top: 0.5rem;
    right: 1.75rem;
    text-align: end;
    color: var(--accent-color);
    font-size: min(8vw, 5vh);

    &:hover {
      cursor: pointer;
    }
  }
}

// From Sara Soueidan (https://www.sarasoueidan.com/blog/focus-indicators/) & Erik Kroes (https://www.erikkroes.nl/blog/the-universal-focus-state/)
:focus-visible,
button:focus-visible,
.focus-visible,
.v-selection-control--focus-visible .v-selection-control__input {
  outline: 9px double white !important;
  box-shadow: 0 0 0 6px black !important;
  border-radius: .125rem;
}

.video-wrapper {
  height: 100%;
  background: black;
  text-align: center;
  z-index: 1000;

  #video-close-icon {
    position: absolute;
    top: 10px;
    right: 10px;
    z-index: 15;
    
    &:hover {
      cursor: pointer;
    }

    &:focus {
      color: white;
      border: 2px solid white;
    }
  }
}

video {
  height: 100%;
  width: auto;
  max-width: 100%;
  object-fit: contain;
}

#info-video {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  max-width: 100%;
  overflow: hidden;
  padding: 0px;
  z-index: 10;
}

.bottom-sheet {
  .v-overlay__content {
    align-self: flex-end;
    padding: 0;
    margin: 0;
    max-width: 100%;
    height: 34%;
  }

  #tabs {
    width: calc(100% - 3em);
    align-self: left;
  }
  
  .info-text {
    height: 33vh;
    padding-bottom: 25px;
  
    & a {
      text-decoration: none;
    }
  }
  
  .close-icon {
    position: absolute;
    top: 10px;
    right: 10px;
    z-index: 15;
  
    &:hover {
      cursor: pointer;
    }
  
    &:focus {
      color: white;
      border: 2px solid white;
    }
  }
  
  .scrollable {
    overflow-y: auto;
  }
  
  #tab-items {
    // padding-bottom: 2px !important;
  
    .v-card-text {
      font-size: ~"max(14px, calc(0.7em + 0.3vw))";
      padding-top: ~"max(2vw, 16px)";
      padding-left: ~"max(4vw, 16px)";
      padding-right: ~"max(4vw, 16px)";
  
      .end-spacer {
        height: 25px;
      }
    }
  
  }
  
  #close-text-icon {
    position: absolute;
    top: 0.25em;
    right: calc((3em - 0.6875em) / 3); // font-awesome-icons have width 0.6875em
    color: white;
  }

  // This prevents the tabs from having some extra space to the left when the screen is small
  // (around 400px or less)
  .v-tabs:not(.v-tabs--vertical).v-tabs--right>.v-slide-group--is-overflowing.v-tabs-bar--is-mobile:not(.v-slide-group--has-affixes) .v-slide-group__next, .v-tabs:not(.v-tabs--vertical):not(.v-tabs--right)>.v-slide-group--is-overflowing.v-tabs-bar--is-mobile:not(.v-slide-group--has-affixes) .v-slide-group__prev {
    display: none;
  }
}
</style>
