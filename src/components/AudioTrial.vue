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

    <div v-if="!doneListening" class="audio-controls">
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
      // audio playback
      playCount: 0,
      replayCount: 0,
      endReachedCount: 0,
      fullListenCount: 0,
      
      isPlaying: false,
      hasFinishedOnce: false,
      doneListening: false,
      
      // Was the previous playback sitting at the end?
      endedSinceLastPlay: false,
      
      // Current listening pass
      passActive: false,
      passStartedAtBeginning: false,
      passHadForwardSeek: false,
      
      // Precise playback-position tracking
      lastPlaybackTime: 0,
      playbackTracker: null,
      
      // One logical seek gesture
      seekGestureActive: false,
      seekFrom: null,
      seekTo: null,
      seekStartedAfterEnd: false,
      seekFinalizeTimer: null,
      
      // Recorded seeking behavior
      seekEvents: [],
      backwardSeekCount: 0,
      forwardSeekCount: 0,
    };
  },

  computed: {
    audioSrc() {
      return `${process.env.BASE_URL}audio_files/${this.trial.audio_file}`;
    },
  },

  methods: {
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
    
    onAudioPlay() {
      const audio = this.$refs.audio;
    
      this.isPlaying = true;
      this.playCount += 1;
    
      // If the audio had ended and Play restarted it from 0,
      // count that as a replay, not a manual rewind.
      if (this.endedSinceLastPlay) {
        if (audio.currentTime <= 0.1) {
          this.replayCount += 1;
        }
    
        this.endedSinceLastPlay = false;
      }
    
      // Start a new listening pass only if we are not already in one.
      // Pause/resume should not create a new pass.
      if (!this.passActive) {
        this.passActive = true;
        this.passStartedAtBeginning = audio.currentTime <= 0.1;
        this.passHadForwardSeek = false;
      }
    
      this.startPlaybackTracker();
    },
    
    onAudioPause() {
      this.isPlaying = false;
      this.stopPlaybackTracker();
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
      this.stopPlaybackTracker();
    
      this.hasFinishedOnce = true;
    
      this.endReachedCount += 1;
    
      // A "full listen" means playback began at the beginning
      // and did not skip forward over part of the recording.
      if (
        this.passActive &&
        this.passStartedAtBeginning &&
        !this.passHadForwardSeek
      ) {
        this.fullListenCount += 1;
      }
    
      this.passActive = false;
    
      // Used to detect a later replay from the beginning.
      this.endedSinceLastPlay = true;
    },

    //records every 50 ms instead of timeUpdate so that even you go back a lot fast, it still records it
    startPlaybackTracker() {
      this.stopPlaybackTracker();
    
      this.playbackTracker = setInterval(() => {
        const audio = this.$refs.audio;
    
        if (audio && !this.seekGestureActive) {
          this.lastPlaybackTime = audio.currentTime;
        }
      }, 50);
    },
    
    stopPlaybackTracker() {
      if (this.playbackTracker !== null) {
        clearInterval(this.playbackTracker);
        this.playbackTracker = null;
      }
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


    //where participant is in audio currently
    onTimeUpdate() {
      const audio = this.$refs.audio;
    
      if (!this.isSeeking) {
        this.lastPlaybackTime = audio.currentTime;
      }
    },
    
    //if participant starts dragging, where were they when they started
    onSeeking() {
      const audio = this.$refs.audio;
    
      // Beginning of ONE logical drag gesture
      if (!this.seekGestureActive) {
        this.seekGestureActive = true;
    
        this.seekFrom = this.lastPlaybackTime;
    
        // Remember whether the audio had already ended.
        // This lets us recognize the browser automatically
        // resetting end -> 0 when Replay is pressed.
        this.seekStartedAfterEnd = this.endedSinceLastPlay;
      }
    
      // During dragging this may update many times.
      // We keep replacing it with the newest destination.
      this.seekTo = audio.currentTime;
    },
    
    //when dragging, stores from and to times in audio recording
    onSeeked() {
      const audio = this.$refs.audio;
    
      this.seekTo = audio.currentTime;
    
      // Browsers can fire many seek events while one slider drag
      // is happening. Wait briefly to see whether more arrive.
      clearTimeout(this.seekFinalizeTimer);
    
      this.seekFinalizeTimer = setTimeout(() => {
        this.finalizeSeek();
      }, 200);
    },

    //makes sure that one long seek back is not counted as many small ones -- like from 3.6 to 3.2, from 3.2 to 2.9, from 2.9 etc.
    finalizeSeek() {
      const from = this.seekFrom;
      const to = this.seekTo;
    
      if (from !== null && to !== null) {
    
        // If the recording had ended and the browser automatically
        // jumped from the end back to 0 when Play was pressed,
        // do NOT count that as a manual seek.
        const automaticReplayReset =
          this.seekStartedAfterEnd &&
          to <= 0.05;
    
        if (!automaticReplayReset) {
          const change = to - from;
    
          // Ignore tiny timing noise
          if (Math.abs(change) >= 0.1) {
            const direction =
              change < 0 ? "backward" : "forward";
    
            const event = {
              from: Number(from.toFixed(3)),
              to: Number(to.toFixed(3)),
              direction: direction,
              amount: Number(Math.abs(change).toFixed(3)),
              trialTime: Date.now() - this.trialStart,
            };
    
            this.seekEvents.push(event);
    
            if (direction === "backward") {
              this.backwardSeekCount += 1;
            } else {
              this.forwardSeekCount += 1;
              this.passHadForwardSeek = true;
            }
    
            // If they seek all the way back to the beginning,
            // a complete listen can begin from here.
            if (to <= 0.1) {
              this.passStartedAtBeginning = true;
              this.passHadForwardSeek = false;
            }
          }
        }
      }
    
      this.seekGestureActive = false;
      this.seekFrom = null;
      this.seekTo = null;
      this.seekStartedAfterEnd = false;
    
      if (this.$refs.audio) {
        this.lastPlaybackTime =
          this.$refs.audio.currentTime;
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
