<template>
  <v-expansion-panel ref="panel">
    <v-expansion-panel-header ref="header">
      <div class="wave" ref="waveform"></div>
      <div class="float-left buttons">
        <v-btn left
               fab
               small
               class="mx-2"
               :disabled="recordDisabled"
               :color="isRecording ? 'red':''"
               @click.stop="recordClickHandler">
          <v-icon>fas fa-{{ isRecording ? 'stop' : 'microphone' }}</v-icon>
        </v-btn>
        <v-btn left
               fab
               small
               color="blue"
               class="mx-2"
               :disabled="playDisabled"
               @click.stop="play">
          <v-icon>fas fa-{{ isPlaying ? 'stop' : 'play' }}</v-icon>
        </v-btn>
      </div>
    </v-expansion-panel-header>
    <v-expansion-panel-content>{{ prompt.phrase }}</v-expansion-panel-content>
  </v-expansion-panel>
</template>

<script>
  import WaveSurfer from 'wavesurfer.js'

  const _globalState = {
    isRecording: false,
    isPlaying: false,
  };

  export default {
    name: "AudioPrompt",
    props: {
      value: {
        type: Object,
        required: true,
      },
      stream: MediaStream
    },

    data() {
      return {
        isRecording: false,
        isPlaying: false,
        globalState: _globalState,
        recorder: null,
        waveSurfer: null,
        prompt: this.value
      }
    },

    methods: {
      recordClickHandler() {
        if (this.isRecording) {
          this.stopRecording();
          this.isRecording = false;
        } else {
          this.record();

          // Open this element if needed.
          if (!this.$refs.panel.isActive) {
            this.$refs.header.$el.click();
          }

          this.isRecording = true;
        }
      },

      record() {
        this.recorder = new MediaRecorder(this.stream);

        const chunks = [];

        this.recorder.addEventListener('dataavailable', (e) => {
          if (e.data.size > 0) {
            chunks.push(e.data)
          }
        });

        this.recorder.addEventListener('stop', () => {
          const blob = new Blob(chunks);
          const reader = new FileReader();
          reader.onloadend = () => {
            this.prompt.data = `data:${this.recorder.mimeType};base64,${reader.result.split(',')[1]}`;
            this.wavesurfer.load(this.prompt.data);
            this.recorder = null; // Done with this recorder
            this.$emit("input", this.prompt);
          };
          reader.readAsDataURL(blob);
        });

        this.recorder.start();
      },

      stopRecording() {
        this.recorder.stop();
      },

      play() {
        if (this.isPlaying) {
          this.wavesurfer.stop();
          this.isPlaying = false;
        } else {
          this.wavesurfer.play(0);
          this.isPlaying = true;
        }
      }
    },

    created() {
      if (this.prompt.data === undefined) {
        this.prompt.data = null;
      }
    },

    mounted() {
      this.wavesurfer = WaveSurfer.create({
        container: this.$refs.waveform,
        interaction: false,
        height: 40,
        barHeight: 5
      });

      const stopped = () => {
        this.isPlaying = false;
      };

      this.wavesurfer.on('pause', stopped);
      this.wavesurfer.on('stop', stopped);

      if (this.prompt.data) {
        this.wavesurfer.load(this.prompt.data);
      }
    },

    computed: {
      recordDisabled() {
        return this.globalState.isPlaying || (this.globalState.isRecording && !this.isRecording) || this.globalState.stream === null
      },

      playDisabled() {
        return this.globalState.isRecording || (this.globalState.isPlaying && !this.isPlaying) || this.prompt.data === null;
      }
    },

    watch: {
      isPlaying(val) {
        this.globalState.isPlaying = val;
      },

      isRecording(val) {
        this.globalState.isRecording = val;
      }
    }
  }
</script>

<style scoped>
  .wave {
    position: absolute;
    width: 100%;
    height: 40px;
    pointer-events: none;
  }

  .buttons {
    z-index: 2;
  }
</style>
