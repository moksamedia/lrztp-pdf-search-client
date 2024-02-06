<script setup>
import {ref, inject} from 'vue'
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
const pdfHit = ref(null)
const pdfNumPages = ref(0)

function isString(test) {
  return typeof text !== 'string' || text instanceof String
}

function processTibetan(text, style, _class) {
  return text.replace(/([\u0F00-\u0FFF]+\s?|<em>|<\/em>)+/gm, (match) => {
    return `<span class="${_class}">${match}</span>`
  })
}

function styleTibetan(text, searchTerm) {
  if (text == null || !isString(text) || text.trim() === "") return text;
  let processed = processTibetan(text, null, "tibetan")
  return processed
}

async function doSearch() {
  console.log("searchTerm="+searchTerm.value)
  const highlight = {
    fields: {
      text: {}
    },
    boundary_chars:".,!? \t\n།",
    boundary_scanner:'word',
    fragment_size: 40,
    type: 'plain'
  };
  const query = {
    index: ['lrztp','tibetan_pdfs'],
    highlight: highlight,
    query: {
      match_phrase: {
        text: searchTerm.value
      }
    }
  };
  const results = await axios.post('http://localhost:3000', {
    query: query
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
      loadByPages: hit._source.loadbypages,
      numpages: hit._source.numpages, // only filled for loadbypages pdfs
      highlights: hit.highlight.text,
      module: hit._source.module,
      lesson: hit._source.lesson,
      filterKey: `${hit._source.baseName}-${hit._source.page}`
    }
  })

  let hitsFiltered = new Map()

  hitsProcessed.forEach (hit => {
    console.log('hit.filterKey:' + hit.filterKey)
    const hitAlreadyInSet = hitsFiltered.get(hit.filterKey);
    if (hitAlreadyInSet !== undefined) {
      console.log('hitAlreadyInSet:' + hitAlreadyInSet.filterKey)
      //compare score and keep best score
      if (hit.score > hitAlreadyInSet.score) {
        hit.hiddenHit = hitAlreadyInSet
        hitsFiltered.set(hit.filterKey, hit)
      }
    }
    else {
      hitsFiltered.set(hit.filterKey, hit)
    }
  })

  hitsFiltered = Array.from(hitsFiltered.values())
  const hitsSorted = hitsFiltered.sort((a, b) => {
    if (a.filterKey < b.filterKey) {
      return -1;
    }
    if (a.filterKey > b.filterKey) {
      return 1;
    }
    return 0;
  });

  console.log("hitsProcessed",hitsProcessed)
  console.log("hitsSorted",hitsSorted)
  searchHits.value = hitsSorted
}
function showPdf(hit) {
  console.log(`Showing pdf ${hit.pdfFile} page ${hit.page}`,hit);
  pdfPage.value = hit.page
  pdfHit.value = hit
  if (hit.loadByPages) {
    pdfNumPages.value = hit.numpages
    pdfSource.value = encodeURI(`./pdfs/${hit.pdfFile}/${hit.pdfFile}_ocr_pp${hit.page}.pdf`)
  }
  else {
    pdfSource.value = encodeURI(`./pdfs/${hit.pdfFile}.pdf`)
  }
}
function previousPage() {
  if (pdfPage.value === 1) return;
  pdfPage.value = pdfPage.value - 1
  if (pdfHit.value.loadByPages) {
    pdfSource.value = encodeURI(`./pdfs/${pdfHit.value.pdfFile}/${pdfHit.value.pdfFile}_ocr_pp${pdfPage.value}.pdf`)
  }
}
function nextPage() {
  if (pdfPage.value === pdfNumPages.value) return;
  pdfPage.value = pdfPage.value + 1
  if (pdfHit.value.loadByPages) {
    pdfSource.value = encodeURI(`./pdfs/${pdfHit.value.pdfFile}/${pdfHit.value.pdfFile}_ocr_pp${pdfPage.value}.pdf`)
  }
}

function handlePdfLoad({numPages}) {
  if (pdfHit.value.loadByPages) return;
  pdfNumPages.value = numPages
  //find(aString, aCaseSensitive, aBackwards, aWrapAround, aWholeWord, aSearchInFrames, aShowDialog)
  //theWindow.find(searchTerm.value,false,false,false,false,true,true)
}

function goBack() {
  pdfPage.value = 1
  pdfHit.value = null
  pdfSource.value = ''
}

</script>
<template>
  <v-container>
    <div v-if="pdfHit">
      <v-row>
        <v-col>
          <div style="margin-bottom: 10px"><v-btn rounded="lg" @click="goBack">Back to results list</v-btn></div>
          <div><h2>{{pdfHit.pdfFile}}.pdf, page {{pdfHit.page}} {{pdfHit.ocr ? "ocr" : "raw"}}</h2> <span style="float: right">(score {{pdfHit.score}})</span></div>
          <div>Module {{pdfHit.module}}, Lesson {{pdfHit.lesson}}</div>
          <ul class="highlight-list-container">
            <li class="highlight" v-for="(highlight,j) in pdfHit.highlights" :key="highlight+j" v-html="styleTibetan(highlight, searchTerm)"></li>
          </ul>
        </v-col>
      </v-row>
      <v-row>
        <div style="width: 1000px; margin: 30px auto 0px;" id="pdf-viewer-container">
          <v-btn :disabled="pdfPage === 1" rounded="lg" size="small" @click="previousPage">prev</v-btn>
          <v-btn :disabled="pdfPage === pdfNumPages" rounded="lg" size="small" @click="nextPage">next</v-btn>
          <span>Showing page {{pdfPage}} of {{pdfNumPages}}</span>
          <VuePdfEmbed
              id="pdf-viewer"
              :source="pdfSource"
              :page="pdfHit.loadByPages ? 1 : pdfPage"
              text-layer annotation-layer width="1000"
              @loaded="handlePdfLoad"
          />
        </div>
      </v-row>
    </div>
    <div v-else>
      <v-row>
        <v-col>
          <v-text-field
              class="search-text"
              label="Search"
              append-icon="mdi-magnify"
              variant="outlined"
              v-model="searchTerm"
              @click:append="doSearch"
          ></v-text-field>
        </v-col>
      </v-row>
      <v-row>
        <v-col class="search-results-col">
          <v-row v-for="(hit, i) in searchHits" :key="hit+i">
            <v-col @click="showPdf(hit)">
              <div><h2>{{hit.pdfFile}}.pdf, page {{hit.page}} {{hit.ocr ? "ocr" : "raw"}}</h2> <span style="float: right">(score {{hit.score}})</span></div>
              <div>Module {{hit.module}}, Lesson {{hit.lesson}}</div>
              <ul class="highlight-list-container">
                <li class="highlight" v-for="(highlight,j) in hit.highlights" :key="hit+i+highlight+j" v-html="styleTibetan(highlight, searchTerm)"></li>
              </ul>
            </v-col>
          </v-row>
        </v-col>
      </v-row>
    </div>
  </v-container>
</template>
<style>
.v-container {
  margin-top: 50px;
}
.search-text {
  max-width: 600px;
}
.tibetan {
  font-size: 160%;
}
.highlight em, .highlight .tibetan em {
  background-color: yellow !important;
  font-style: normal !important;
}
.search-results-col {
}
#pdf-viewer {
}
#pdf-viewer-container .v-btn {
  margin: 10px;
}
#pdf-viewer-container {
  text-align: center;
}
.highlight-list-container ul.highlight li {
  list-style-type: circle;
  color: black;
  list-style-position: inside;
  margin: 0px;
  padding: 0px;
}
.highlight-list-container {
  margin-left: 20px;
}

</style>
