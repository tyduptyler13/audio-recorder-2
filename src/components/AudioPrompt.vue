<template>
  <div>
    <v-expansion-panel-header>
      <v-btn absolute left icon text color="red"><v-icon>fas fa-microphone</v-icon></v-btn>
    </v-expansion-panel-header>
    <v-expansion-panel-content>{{ prompt.phrase }}</v-expansion-panel-content>
  </div>
</template>

<script>
  export default {
    name: "AudioPrompt",
    props: {
      prompt: {
        phrase: String,
        fileName: String
      }
    }
  }
</script>
