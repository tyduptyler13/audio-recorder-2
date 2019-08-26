<template>
  <div>
    <v-file-input type="file"
                  accept="text/csv"
                  label="Prompts CSV"
                  v-model="promptsCSV"></v-file-input>
  </div>
</template>

<script>
  function collectMatches(regex, input) {
    let match, result = [];
    while ((match = regex.exec(input)) !== null) {
      result.push(match);
    }
    return result;
  }

  export default {
    name: "PromptInput",

    data() {
      return {
        promptsCSV: null,
        prompts: null
      }
    },

    watch: {
      promptsCSV(promptsCSV) {
        const lineFinder = /(?:(?:"(?:[^"]|"")*"|[^",\r\n]+)(?:,|$))+/gm;
        const cellFinder = /(?:"((?:[^"]|"")*)"|([^",\r\n]+)|(?<=,))(?:,|$)/gm; // (Quoted, regular, empty) (followed by a comma or end of line)
        if (promptsCSV) {

          const reader = new FileReader();

          const handleCSV = this.handleCSV;
          // Csv parser
          reader.onload = function() {
            handleCSV( // Return the csv as a 2d array
                collectMatches(lineFinder, this.result).map(row => { // get every line
                  return collectMatches(cellFinder, row[0]).map(cell => { // get every cell
                    return cell[1] || cell[2] || ""; // Only get the capture groups or use an empty string as backup
                  }).map(cell => {
                    return cell.replace('""', '"') // Unescape any remaining quotes
                  });
                }));
          };

          reader.readAsText(promptsCSV);
        }
      }
    },
    methods: {
      handleCSV(rawPrompts) {
        const headers = rawPrompts[0].map(header => header.toLowerCase()); // first row, to lower case names
        const prompts = rawPrompts.slice(1).map(row => { // For every row (skipping the first row)
          // Stuff every row into an object containing the rows with their header rows so they can be accessed by name
          const ret = {};
          for (let i = 0; i < headers.length; ++i) {
            ret[headers[i]] = row[i];
          }
          return ret;
        }).filter(row => {
          return row.phrase.length !== 0 // Ignore empty phrases
        });

        this.$emit('input', prompts);
      }
    }
  }
</script>

<style scoped>
</style>