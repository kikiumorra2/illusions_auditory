<template>
  <div class = "audio-trial">
    
    <div class = "trial-counter">
      <template v-if="trial.phase === 'practice'">
        Practice Sentence {{ number }}
      </template>

      <template v-else>
        Sentence {{ number }} of {{ total }}
      </template>
    </div>

    <div v-if="!doneListening" class="audio-controls">
      <audio
        ref="audio"
        :src="audioSrc"
        preload="metadata"
        @loadedmetadata="onLoadedMetadata"
        @timeupdate="onTimeUpdate"
        @ended="onAudioEnded"
      ></audio>

      <button @click="togglePlayback">
        {{ isPlaying ? "Pause" : "Play" }}
      </button>
      
    <div class="custom-audio-player">

      <span>
        {{ formatTime(displayTime) }}
      </span>
    
      <input
        type="range"
        min="0"
        :max="duration"
        step="0.01"
        :value="displayTime"
        @pointerdown="startSeek"
        @input="updateSeek"
        @pointerup="finishSeek"
        @change="finishSeek"
        @pointercancel="cancelSeek"
      >
    
      <span>
        {{ formatTime(duration) }}
      </span>
    
    </div>
    
      <div class="done-listening-button">
        <button
          :disabled="!hasFinishedOnce"
          @click="finishListening"
        >
          Done listening
        </button>
      </div>
    </div>
   

    <!-- Only show ratings once "Done Listening" button has been pressed -->
    <div v-if="doneListening" class="ratings">

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
      // Ratings
      grammarRating: null,
      meaningRating: null,
  
      // Timing
      trialStart: Date.now(),
      doneListeningTime: null,
  
      // Audio position
      duration: 0,
      currentTime: 0,
  
      // Playback
      isPlaying: false,
      hasFinishedOnce: false,
      doneListening: false,
  
      playCount: 0,
      replayCount: 0,
      endReachedCount: 0,
      fullListenCount: 0,
  
      // Current listening pass
      passActive: false,
      passStartedAtBeginning: false,
      passHadForwardSeek: false,
  
      // Custom seek slider
      isDragging: false,
      seekFrom: null,
      seekPreview: 0,
      wasPlayingBeforeSeek: false,
  
      // Recorded seeking
      seekEvents: [],
      backwardSeekCount: 0,
      forwardSeekCount: 0,
    };
  },

  computed: {
    audioSrc() {
      return `${process.env.BASE_URL}audio_files/${this.trial.audio_file}`;
    },
  
    displayTime() {
      if (this.isDragging) {
        return this.seekPreview;
      }
  
      return this.currentTime;
    },
  },

  methods: {
    formatTime(seconds) {
      if (!Number.isFinite(seconds)) {
        return "0:00";
      }
  
      const minutes = Math.floor(seconds / 60);
      const secs = Math.floor(seconds % 60);
  
      return `${minutes}:${String(secs).padStart(2, "0")}`;
    },
    
    
    
    
    //makes doneListing --> true after pressing button
    finishListening() {
      const audio = this.$refs.audio;
    
      if (audio) {
        audio.pause();
      }
    
      this.isPlaying = false;
      this.doneListeningTime = Date.now() - this.trialStart;
      this.doneListening = true;
    },
    
   
    
   
    
    onAudioEnded() {
      this.isPlaying = false;
      this.currentTime = this.duration;
    
      this.hasFinishedOnce = true;
      this.endReachedCount += 1;
    
      if (
        this.passActive &&
        this.passStartedAtBeginning &&
        !this.passHadForwardSeek
      ) {
        this.fullListenCount += 1;
      }
    
      this.passActive = false;
    },

   
    
  


    finishTrial() {
      const row = {
        // Experiment information
        Experiment: config.experimentName,
        ListId: this.listId,
    
        // Trial information
        Condition: this.trial.condition_id,
        ItemId: this.trial.item_id,
        TrialId: this.index,
        TrialType: "audio_trial",
        Phase: this.trial.phase,
    
        TrialText: this.trial.text,
        AudioFile: this.trial.audio_file,
    
        // Ratings
        grammarRating: this.grammarRating,
        meaningRating: this.meaningRating,
    
        // Listening behavior
        playCount: this.playCount,
        replayCount: this.replayCount,
    
        // Number of times playback reached the end
        endReachedCount: this.endReachedCount,
    
        // Number of complete beginning-to-end listens
        fullListenCount: this.fullListenCount,
    
        // Seeking behavior
        backwardSeekCount: this.backwardSeekCount,
        forwardSeekCount: this.forwardSeekCount,
    
        // Full details of every seek
        seekEvents: JSON.stringify(this.seekEvents),
    
        // Timing
        doneListeningTime: this.doneListeningTime,
        trialTime: Date.now() - this.trialStart,
      };
    
      // Helpful while testing
      console.log("RECORDED AUDIO TRIAL:", row);
      console.log("SEEK EVENTS:", this.seekEvents);
    
      // Add the trial to Magpie's data
      this.$magpie.addTrialData(row);
    
      // Show everything recorded so far in the browser console
      console.table(this.$magpie.getAllData());
    
      // Send this trial immediately if configured to do so
      if (config.submitEachTrial) {
        this.submitTrial();
      }
    
      // Tell App.vue that this trial is finished
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

    onLoadedMetadata() {
      const audio = this.$refs.audio;
    
      this.duration = audio.duration;
      this.currentTime = audio.currentTime;
    },

    startSeek(event) {
      const audio = this.$refs.audio;
    
      event.target.setPointerCapture(event.pointerId);
    
      this.isDragging = true;
    
      this.seekFrom = audio.currentTime;
      this.seekPreview = audio.currentTime;
    
      this.wasPlayingBeforeSeek = !audio.paused;
    
      if (this.wasPlayingBeforeSeek) {
        audio.pause();
        this.isPlaying = false;
      }
    },

    //doesn't record, just updates and makes one long seek instead of many small ones
    updateSeek(event) {
      this.seekPreview = Number(event.target.value);
    },

    finishSeek(event) {
      // pointerup and change can both fire.
      // Only process the first one.
      if (!this.isDragging) {
        return;
      }
    
      const audio = this.$refs.audio;
    
      const from = this.seekFrom;
      const to = Number(event.target.value);
    
      // Move the actual recording
      audio.currentTime = to;
      this.currentTime = to;
    
      const change = to - from;
    
      if (Math.abs(change) >= 0.01) {
        const direction =
          change < 0 ? "backward" : "forward";
    
        const eventData = {
          from: Number(from.toFixed(3)),
          to: Number(to.toFixed(3)),
          direction: direction,
          amount: Number(Math.abs(change).toFixed(3)),
          trialTime: Date.now() - this.trialStart,
        };
    
        this.seekEvents.push(eventData);
    
        if (direction === "backward") {
          this.backwardSeekCount += 1;
        } else {
          this.forwardSeekCount += 1;
    
          // Forward skipping means this wasn't a complete listen
          this.passHadForwardSeek = true;
        }
    
        // Seeking back to beginning can start a complete listen
        if (to <= 0.05) {
          this.passActive = true;
          this.passStartedAtBeginning = true;
          this.passHadForwardSeek = false;
        }
      }
    
      this.isDragging = false;
      this.seekFrom = null;
      this.seekPreview = to;
    
      // Resume automatically if it was playing before the seek
      if (this.wasPlayingBeforeSeek) {
        audio.play()
          .then(() => {
            this.isPlaying = true;
          })
          .catch((error) => {
            console.error("Could not resume audio:", error);
          });
      }
    
      this.wasPlayingBeforeSeek = false;
    },
    
    cancelSeek() {
      this.isDragging = false;
      this.seekFrom = null;
      this.wasPlayingBeforeSeek = false;
    },
    
    async togglePlayback() {
      const audio = this.$refs.audio;
    
      // PAUSE
      if (this.isPlaying) {
        audio.pause();
        this.isPlaying = false;
        return;
      }
    
      // If audio has reached the end, Play means replay from beginning
      if (
        audio.ended ||
        (
          this.duration > 0 &&
          audio.currentTime >= this.duration - 0.05
        )
      ) {
        audio.currentTime = 0;
        this.currentTime = 0;
    
        this.replayCount += 1;
    
        this.passActive = true;
        this.passStartedAtBeginning = true;
        this.passHadForwardSeek = false;
      }
    
      // Otherwise start a new listening pass if needed
      else if (!this.passActive) {
        this.passActive = true;
    
        this.passStartedAtBeginning =
          audio.currentTime <= 0.05;
    
        this.passHadForwardSeek = false;
      }
    
      try {
        await audio.play();
    
        this.isPlaying = true;
        this.playCount += 1;
    
      } catch (error) {
        console.error("Could not play audio:", error);
      }
    },

    
    //where participant is in audio currently
    onTimeUpdate() {
      const audio = this.$refs.audio;
    
      if (!this.isDragging) {
        this.currentTime = audio.currentTime;
      }
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

.custom-audio-player {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 12px;

  width: 100%;
  max-width: 600px;

  margin: 20px auto;
}

.custom-audio-player input[type="range"] {
  flex: 1;
}

.done-listening-button {
  margin-top: 20px;
}
</style>
