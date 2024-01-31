<script setup>
import {ref} from 'vue'
import {useSearchStore} from "../stores/search.js";
import { storeToRefs } from 'pinia'
import axios from 'axios'
import _ from 'lodash'
import VuePdfEmbed from 'vue-pdf-embed'
// essential styles
import 'vue-pdf-embed/dist/style/index.css'
// optional styles
import 'vue-pdf-embed/dist/style/annotationLayer.css'
import 'vue-pdf-embed/dist/style/textLayer.css'

const searchStore = useSearchStore()
const { searchTerm } = storeToRefs(searchStore)
const searchHits = ref([])

const pdfSource = ref('')
const pdfPage = ref(1)

function isString(test) {
  return typeof text !== 'string' || text instanceof String
}

function processTibetan(text, style, _class) {
  return text.replace(/([\u0F00-\u0FFF]+|<em>|<\/em>)+/gm, (match) => {
    return `<span style="${style}" class="${_class}">${match}</span>`
  })
}

function highlight(words, query, highlightClass) {

  if (!words || typeof words !== 'string') return words;

  query = query.replace(/"/g, "");

  console.log("words="+words)
  console.log("query="+query)

  let spanOpen = '<span class="'+highlightClass+'">';
  let spanClose = '</span>';

  return words.replace(new RegExp(query, "g"), function (match) {
    return spanOpen+match+spanClose;
  });

}

function styleTibetan(text, searchTerm) {
  if (text == null || !isString(text) || text.trim() === "") return text;
  let processed = processTibetan(text, null, "tibetan")
  return processed
}

async function doSearch() {
  console.log("searchTerm="+searchTerm.value)
  const results = await axios.get('http://localhost:3000', {
    params: {
      searchTerm: searchTerm.value
    }
  })
  console.log(results)
  //console.log(JSON.stringify(results,null,2))
  const numHits = _.get(results, 'data.result.hits.total.value', null)
  const millis = _.get(results, 'data.results.took', 0)
  const hits = _.get(results, 'data.result.hits.hits', null)
  console.log(`${numHits} hits in ${millis}ms:`, hits)
  const filterSet = new Set()
  const hitsProcessed = hits.map(hit => {
    return {
      score: hit._score,
      pdfFile: hit._source.baseName,
      searchFile: hit._source.fileName,
      ocr: hit._source.ocr,
      page: hit._source.page,
      highlights: hit.highlight.text
    }
  })
  const hitsFiltered = hitsProcessed.filter(hit => {
    const filterKey = `${hit.pdfFile}-${hit.page}`
    if (!filterSet.has(filterKey)) {
      filterSet.add(filterKey)
      return true
    }
    else {
      return false
    }
  })
  console.log("hitsProcessed",hitsProcessed)
  console.log("hitsFiltered",hitsFiltered)
  searchHits.value = hitsFiltered
}
function showPdf(hit) {
  console.log(`Showing pdf ${hit.pdfFile} page ${hit.page}`)
  pdfPage.value = hit.page
  pdfSource.value = encodeURI(`./pdfs/${hit.pdfFile}.pdf`)
}
</script>
<template>
  <v-container fluid>
    <v-row no-gutters>
      <v-col>
        <v-text-field
            class="search-text"
            label="Label"
            append-icon="mdi-magnify"
            variant="outlined"
            v-model="searchTerm"
            @click:append="doSearch"
        ></v-text-field>
      </v-col>
    </v-row>
    <v-row v-for="(hit, i) in searchHits" :key="hit+i">
      <v-col @click="showPdf(hit)">
        {{hit.pdfFile}}.pdf, page {{hit.page}} <span style="float: right">(score {{hit.score}})</span>
        <div class="highlight" v-for="(highlight,j) in hit.highlights" :key="hit+i+highlight+j" v-html="styleTibetan(highlight, searchTerm)"></div>
      </v-col>
    </v-row>
    <v-row v-if="pdfSource" justify="center">
      <v-col align-self="center" cols="8">
        <VuePdfEmbed :source="pdfSource" :page="pdfPage" text-layer annotation-layer width="800"/>
      </v-col>
    </v-row>
  </v-container>
</template>
<style>
.v-container {
  margin-top: 50px;
}
.search-text {
}
.tibetan {
  font-size: 160%;
}
.highlight em, .highlight .tibetan em {
  background-color: yellow !important;
  font-style: normal !important;
}
</style>
