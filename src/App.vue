<template>
  <v-app>
    <v-app-bar app>
      <v-toolbar-title class="headline text-uppercase">
        <span>Audio Recorder 2</span>
        <span class="font-weight-light">The web edition</span>
      </v-toolbar-title>
      <v-spacer></v-spacer>
      <v-fab-transition v-if="canSave">
        <v-btn fab
               absolute
               bottom
               right
               color="green"
               @click="save"
               :loading="saving">
          <v-icon>fa fa-save</v-icon>
        </v-btn>
      </v-fab-transition>
    </v-app-bar>

    <v-content>
      <prompt-input v-model="prompts"></prompt-input>
      <div v-if="prompts !== null && prompts.length !== 0">
        <v-alert type="warning" v-if="typeof prompts[0].phrase !== 'string' || typeof prompts[0].filename !== 'string'">WARNING: The file
          that was uploaded does not contain the required headers! (Both phrase and filename are required)
        </v-alert>
        <v-alert type="info" dismissible>For each of the following, press record when you are ready and read the text shown aloud. When a
          recording is made, you can play it back to see if it sounds correct or if you should try again. When you are done you can click
          save and export the final result.
        </v-alert>
        <v-expansion-panels>
          <audio-prompt :value="prompt"
                        :stream="globalState.stream"
                        :key="prompt.phrase"
                        @input="updatedPrompt"
                        v-for="prompt in prompts"></audio-prompt>
        </v-expansion-panels>
      </div>
    </v-content>
  </v-app>
</template>

<script>
  import PromptInput from '@/components/PromptInput';
  import AudioPrompt from '@/components/AudioPrompt';
  import JSZip from 'jszip';
  import saveAs from 'file-saver';

  import Vue from 'vue';

  const _globalState = Vue.observable({
    stream: null
  });

  navigator.mediaDevices.getUserMedia({audio: true})
      .then((stream) => {
        _globalState.stream = stream;
      });

  export default {
    name: 'app',
    components: {
      AudioPrompt,
      PromptInput
    },

    data() {
      return {
        prompts: null,
        globalState: _globalState,
        saving: false,
      }
    },

    mounted() {
      if (localStorage.prompts) {
        this.prompts = JSON.parse(localStorage.prompts);
      }
    },

    watch: {
      prompts(prompts) {
        localStorage.prompts = JSON.stringify(prompts);
      }
    },

    methods: {
      updatedPrompt(updatedPrompt) {
        for (let i = 0; i < this.prompts.length; ++i) {
          let prompt = this.prompts[i];
          if (prompt.phrase === updatedPrompt.phrase) {
            prompt.data = updatedPrompt.data;
            break;
          }
        }

        localStorage.prompts = JSON.stringify(this.prompts);
      },

      save() {
        this.saving = true;

        const zip = new JSZip();
        const extensionFinder = /audio\/([a-z]+)/;

        const content = this.prompts.filter(prompt => {
          return prompt.data !== undefined && prompt.data != null
        }).map(prompt => {
          const extension = prompt.data.match(extensionFinder)[1];
          const filename = prompt.filename.split('.')[0] + "." + extension;
          const metaDataPos = prompt.data.indexOf('base64,') + 'base64,'.length;
          zip.file(filename, prompt.data.substring(metaDataPos), {base64: true});
          return {phrase: prompt.phrase, filename: filename};
        });

        zip.file("content.csv", "filename,phrase\r\n" +
            content.map(phrase => {
              return `"${phrase.filename}","${phrase.phrase}"`
            }).join("\r\n") + "\r\n"
        );

        zip.generateAsync({type: "blob"}).then(zip => {
          saveAs(zip, "phrases.zip");
          this.saving = false;
        });
      },

      canSave() {
        return this.prompts !== null && this.prompts.length > 0 && this.prompts.some(prompt => {
          return prompt.data !== null && prompt.data !== undefined
        });
      }
    }
  }
</script>
