<template>
  <div class = "audio-trial">
    
    <div class = "trial-counter">
      <template v-if="trial.phase === 'practice'">
        Practice Sentece {{ number }}
      </template>

      <template v-else>
        Sentence {{ number }} of {{ total }}
      </template>
    </div>

    <!-- Hidden audio player -->
    <audio
      ref = "audio"
      :src="audioSrc"
      preload="auto"
      @ended="onAudioEnded"
    ></audio>

    <!-- Play/Replay button -->
    <div class="audio-controls">
      <button
        :disabled = "isPlaying"
        @click="playAudio"
      >
        {{ playCount === 0 ? "Play sentence" : "Replay sentence" }}
      </button>
    </div>

    <!-- Only show ratings once sentence has been heard completely once -->
    <div v-if="hasFinishedOnce" class="ratings">

      <div class="rating-question">
        <p>
          <b>How grammatically well-formed does this sentence sound to you?</b>
        </p>

        <div class="scale-labels">
          <span>Not at all well-formed</span>
          <span>Completely well-formed</span>
        </div>

        <div class="scale">
          <label
            v-for="n in 7"
            :key="'grammar-' + n"
          >
            <input
              v-model.number="grammarRating"
              type="radio"
              :value="n"
            >
            {{ n }}
          </label>
        </div>
      </div>


      <div class="rating-question">
        <p>
          <b>How much sense does this sentence make to you?</b>
        </p>

        <div class="scale-labels">
          <span>Makes no sense at all</span>
          <span>Makes complete sense</span>
        </div>

        <div class="scale">
          <label
            v-for="n in 7"
            :key="'meaning-' + n"
          >
            <input
              v-model.number="meaningRating"
              type="radio"
              :value="n"
            >
            {{ n }}
          </label>
        </div>
      </div>


      <button
        :disabled="grammarRating === null ||
                   meaningRating === null ||
                   isPlaying"
        @click="finishTrial"
      >
        Next
      </button>
    </div>
  </div>
</template>

<script>
  import config from "../config";
  import { submitRows } from "../submit";

  export default {
  name: "AudioTrial",

  props: {
    trial: {
      type: Object,
      required: true,
    },

    index: {
      type: Number,
      required: true,
    },

    number: {
      type: Number,
      required: true,
    },

    total: {
      type: Number,
      required: true,
    },

    listId: {
      type: [Number, String],
      default: null,
    },
  },

  data() {
    return {
      playCount: 0,
      replayCount: 0,

      isPlaying: false,
      hasFinishedOnce: false,

      grammarRating: null,
      meaningRating: null,

      trialStart: Date.now(),
    };
  },

  computed: {
    audioSrc() {
      return `${process.env.BASE_URL}audio/${this.trial.audio_file}`;
    },
  },

  methods: {
    async playAudio() {
      const audio = this.$refs.audio;

      // Always begin from the start
      audio.currentTime = 0;

      try {
        await audio.play();

        this.playCount += 1;

        // First play is not a replay
        if (this.playCount > 1) {
          this.replayCount += 1;
        }

        this.isPlaying = true;

      } catch (error) {
        console.error("Could not play audio:", error);
      }
    },


    onAudioEnded() {
      this.isPlaying = false;
      this.hasFinishedOnce = true;
    },


    finishTrial() {
      const row = {
        Experiment: config.experimentName,

        Condition: this.trial.condition_id,
        ItemId: this.trial.item_id,

        TrialId: this.index,
        TrialType: "audio_trial",
        Phase: this.trial.phase,

        TrialText: this.trial.text,
        AudioFile: this.trial.row.audio_file,

        grammarRating: this.grammarRating,
        meaningRating: this.meaningRating,

        playCount: this.playCount,
        replayCount: this.replayCount,

        trialTime: Date.now() - this.trialStart,

        ListId: this.listId,
      };

      this.$magpie.addTrialData(row);

      if (config.submitEachTrial) {
        this.submitTrial();
      }

      this.$emit("done");
    },
    
    
    
    submitTrial() {
      const rows = this.$magpie
        .getAllData()
        .filter(
          (r) =>
            String(r.ItemId) === String(this.trial.item_id) &&
            String(r.Condition) === String(this.trial.condition_id)
        );

      submitRows(
        this.$magpie,
        rows,
        `audio trial ${this.trial.item_id}`
      ).catch(() => {});
    },
  },
};
</script>


<style>
.audio-trial {
  width: 50em;
  max-width: 90%;
  margin: 0 auto;
  text-align: center;
}

.trial-counter {
  margin-bottom: 40px;
}

.audio-controls {
  margin: 30px 0;
}

.ratings {
  margin-top: 40px;
}

.rating-question {
  margin: 40px auto;
  max-width: 700px;
}

.scale {
  display: flex;
  justify-content: space-between;
  margin-top: 8px;
}

.scale label {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.scale-labels {
  display: flex;
  justify-content: space-between;
  font-size: 14px;
}
</style>
