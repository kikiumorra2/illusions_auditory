<!--
  BrowserCheck — asks the participant to set the browser zoom to 100% before the study.
  The MoTR window and the word boxes are defined in CSS pixels, so a zoomed page changes
  their physical size on screen. The check re-runs whenever the window is resized (zoom
  changes fire a resize event). Settings: config.browserCheck.
-->
<template>
  <div class="browser-check">

    <!-- STEP 1 -->
    <template v-if="!showPreview">
      <p>
        <b>Please reset your browser zoom to 100% before continuing.</b>
      </p>

      <p>
        Press <kbd>Ctrl</kbd> + <kbd>0</kbd> on Windows/Linux,
        or <kbd>&#8984;</kbd> + <kbd>0</kbd> on Mac.
      </p>

      <p>
        Please use a desktop or laptop computer with a mouse or trackpad,
        keep this window open and do not change the zoom until the study is complete.
      </p>

      <button @click="showPreview = true">
        I have reset the zoom to 100%
      </button>
    </template>

    <!-- STEP 2 -->
    <template v-else>
      <p>
        <b>Please check the sentence below.</b>
      </p>

      <p>
        If the entire blurred sentence does not fit on one line,
        please widen your browser window until it does.
      </p>

      <div class="sentence-preview-window">
        <div
          class="sentence-preview"
          :style="{ fontSize: sentenceFontSize + 'px' }"
        >
          {{ previewSentence }}
        </div>
      </div>

      <p>
        Once the entire sentence fits in the window, you may continue.
      </p>

      <button @click="$emit('done')">
        Continue
      </button>
    </template>

  </div>
</template>


<script>

export default {
  name: "BrowserCheck",

  props: {
    longestSentence: {
      type: String,
      required: true,
    },

    sentenceFontSize: {
      type: Number,
      required: true,
    },
  },

  data() {
    return {
      showPreview: false,
    };
  },

  computed: {
    previewSentence() {
      return "If you do not see this entire text in one line on your screen, please widen your window until it fits.--Please do this before you start.--Thank you!";  
    },
  },
};
</script>

<style>
.browser-check kbd {
  border: 1px solid #999;
  border-radius: 3px;
  padding: 0 4px;
  font-family: inherit;
}

.sentence-preview-window {
  width: 90vw;
  max-width: 90vw;

  margin-top: 30px;
  margin-bottom: 30px;

  margin-left: 50%;
  transform: translateX(-50%);

  overflow: hidden;
  box-sizing: border-box;

  border: 1px solid #999;
  padding: 20px;
}

.sentence-preview {
  font-family: Consolas, monospace;
  font-weight: 450;

  white-space: nowrap !important;
  width: max-content;

  pointer-events: none;

  user-select: none;
  -webkit-user-select: none;
  -moz-user-select: none;
}
</style>
