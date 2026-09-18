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

    <div class="audio-controls">
      <audio
        ref="audio"
        :src="audioSrc"
        controls
        preload="auto"
        @play="onAudioPlay"
        @pause="onAudioPause"
        @ended="onAudioEnded"
        @timeupdate="onTimeUpdate"
        @seeking="onSeeking"
        @seeked="onSeeked"
      ></audio>
    
      <div class="replay-button">
        <button
          v-if="playCount > 0"
          @click="replayAudio"
        >
          Replay from beginning
        </button>
      </div>
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
  
      // seeking information
      lastPlaybackTime: 0,
      seekFrom: null,
      isSeeking: false,
  
      backwardSeekCount: 0,
      backwardSeekEvents: [],
  
      // lets us distinguish participant seeking from our Replay button
      programmaticSeek: false,
    };
  },

  computed: {
    audioSrc() {
      return `${process.env.BASE_URL}audio_files/${this.trial.audio_file}`;
    },
  },

  methods: {
    onAudioPlay() {
      this.isPlaying = true;
    
      // Count the first time the audio is played
      if (this.playCount === 0) {
        this.playCount = 1;
      }
    },
    
    onAudioPause() {
      this.isPlaying = false;
    },
    
    //does not count as dragging
    replayAudio() {
      const audio = this.$refs.audio;
    
      this.programmaticSeek = true;
      audio.currentTime = 0;
    
      this.playCount += 1;
      this.replayCount += 1;
    
      audio.play().catch((error) => {
        console.error("Could not replay audio:", error);
      });
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
        AudioFile: this.trial.audio_file,
      
        grammarRating: this.grammarRating,
        meaningRating: this.meaningRating,
      
        playCount: this.playCount,
        replayCount: this.replayCount,
      
        backwardSeekCount: this.backwardSeekCount,
        backwardSeekEvents: JSON.stringify(this.backwardSeekEvents),
      
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


    //where participant is in audio currently
    onTimeUpdate() {
      const audio = this.$refs.audio;
    
      if (!this.isSeeking) {
        this.lastPlaybackTime = audio.currentTime;
      }
    },
    
    //if participant starts dragging, where were they when they started
    onSeeking() {
      if (this.programmaticSeek) {
        return;
      }
    
      this.isSeeking = true;
    
      // Position before the participant moved the playhead
      this.seekFrom = this.lastPlaybackTime;
    },
    
    //when dragging, stores from and to times in audio recording
    onSeeked() {
      const audio = this.$refs.audio;
    
      if (this.programmaticSeek) {
        this.programmaticSeek = false;
        this.isSeeking = false;
        this.lastPlaybackTime = audio.currentTime;
        return;
      }
    
      const from = this.seekFrom;
      const to = audio.currentTime;
    
      // Only count actual backwards movements.
      // The 0.1 prevents tiny browser timing differences from counting.
      if (from !== null && to < from - 0.1) {
        this.backwardSeekCount += 1;
    
        this.backwardSeekEvents.push({
          from: Number(from.toFixed(3)),
          to: Number(to.toFixed(3)),
          amountBack: Number((from - to).toFixed(3)),
          trialTime: Date.now() - this.trialStart,
        });
      }
    
      this.isSeeking = false;
      this.seekFrom = null;
      this.lastPlaybackTime = to;
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

.audio-controls audio {
  width: 100%;
  max-width: 500px;
}

.replay-button {
  margin-top: 15px;
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
