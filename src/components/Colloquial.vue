<script setup>
import {ref, inject} from 'vue'
import {useSearchStore} from "../stores/search.js";
import { storeToRefs } from 'pinia'
import axios from 'axios'
import _ from 'lodash'

// essential styles
import 'vue-pdf-embed/dist/style/index.css'
// optional styles
import 'vue-pdf-embed/dist/style/annotationLayer.css'
import 'vue-pdf-embed/dist/style/textLayer.css'

const searchStore = useSearchStore()
const { searchTerm } = storeToRefs(searchStore)
const searchHits = ref([])
const searchMaxHits = ref(200)
const numHits = ref(0)
const millis = ref(null)

function isString(test) {
  return typeof text !== 'string' || text instanceof String
}

function processTibetan(text, style, _class) {
  return text.replace(/([\u0F00-\u0FFF]+\s?|<em>|<\/em>)+/gm, (match) => {
    return `<span class="${_class}">${match}</span>`
  })
}

function styleTibetan(text,hit) {
  //console.log(`text=${text}`)

  if (text == null || !isString(text) || text.trim() === "") return text;

  let processed = text

  if (hit && hit._index === 'monlam_tib_tib') {

    // parse numbers "1.", "2.", "3.", etc.
    processed = processed.replace(/([0-9]+)./gm, (match, s1) => {
      if (s1 === "1") return `<span class="circle">${s1}</span>`
      else return `<br/><span class="circle">${s1}</span>`
    })

    processed = processed.replace(/\r\n|\r|\n/gm, "<br/>")

    // monlam seems to use tibetan numbers butted up at front of phrases
    // to mark them for styling
    processed = processed.replace(/[༡-༩]([^\s]*[།ག])/gm, (match, s1) => {
      return ` <span style="color:#6767ce">${s1}</span>`
    })

    processed = processed.replace(/(\s)([༠-༩]{1,2})(\s)/gm, (match, s1, s2, s3) => {
      return `${s1}<span style="color:#6767ce; font-weight:bold">${s2}</span>${s3}`
    })

    processed = processed.replace(/རྒྱུན་སྤྱོད།|དཔེར་ན།/gm, (match) => {
      return `<span style='color: darkgray; font-weight: bold'>${match}</span>`
    });

    processed = processed.replace(/བྱ་ཚིག|མིང་ཚིག|རྒྱན་ཚིག/gm, (match) => {
      return `<div style='color: darkgray; font-weight: bold'>${match}</div>`
    });

  }

  processed = processTibetan(processed, null, "tibetan")

  return processed
}

async function doSearch(more) {
  console.log("searchTerm="+searchTerm.value)
  const highlight = {
    fields: {
      definiendum: {},
      definition: {}
    },
    boundary_chars:".,!? \t\n།",
    number_of_fragments: 0,
    type:'fvh'
  };
  const searchDefinition = true
  const query = searchDefinition ? {
    multi_match: {
      query: searchTerm.value,
      fields: ['definiendum', 'definition']
    }
  } : {
    match: {
      definiendum: searchTerm.value,
    }
  }

  const from = more ? searchHits.value.length : 0

  const results = await axios.post('http://localhost:3000/colloquial', {
    from: from,
    query: query,
    size: searchMaxHits.value,
    index: 'tib_eng_dicts',
    highlight: highlight
  })
  console.log(results)
  //console.log(JSON.stringify(results,null,2))
  numHits.value = _.get(results, 'data.result.hits.total.value', null)
  millis.value = _.get(results, 'data.result.took', 0)
  const hits = _.get(results, 'data.result.hits.hits', null)
  console.log(`${numHits.value} hits in ${millis.value}ms:`, hits)

  if (more) {
    searchHits.value.push(...hits)
  }
  else {
    searchHits.value = hits
  }
}

function loadMore() {

}

function getHighlightIfExists(hit, field) {
  const highlight = _.get(hit, 'highlight.'+field+'[0]', null)
  const fromSource = _.get(hit, '_source.'+field, null)
  //console.log(`highlight=${highlight}, fromSource=${fromSource}`)
  return highlight != null ? highlight : fromSource
}

function getSourceName(index) {
  let name = ""
  switch (index) {
    case 'monlam_tib_eng':
      name = 'Monlam Tibetan-English Dictionary'
      break
    case 'monlam_tib_tib':
      name = 'Monlam Tibetan Dictionary'
      break
    case 'phurba_mahavyutpatti':
      name = 'Mayavyutpatti'
      break
    case 'phurba_rangjung_yeshe':
      name = 'Rangjung Yeshe'
      break
    case 'phurba_tshig_mdzod_chen_mo':
      name = 'Tshig Mzod Chen Mo'
      break
    case 'phurba_new_english_tibetan':
      name = 'New English Tibetan'
      break
    case 'phurba_dag_yig_gsar_bsgrigs':
      name = 'Dag Yig Gsar Bsgrigs'
      break
    case 'phurba_tsultrim_lodro_tib_eng':
      name = 'New Tibetan English'
      break
    case 'phurba_tsultrim_lodro_tib_tib':
      name = 'New Tibetan Tibetan'
      break
  }
  return name
}

</script>
<template>
  <v-container>
    <v-row>
      <v-col>
        <v-text-field
            class="search-text"
            label="Search"
            append-icon="mdi-magnify"
            variant="outlined"
            v-model="searchTerm"
            @click:append="doSearch(false)"
        ></v-text-field>
      </v-col>
    </v-row>
    <v-row v-if="numHits > 0">
      <v-col>
        {{numHits}} hits in {{millis}} milliseconds (showing {{searchHits.length}})
      </v-col>
    </v-row>
    <v-row>
      <v-col class="search-results-col">
        <v-row v-for="(hit,i) in searchHits" :key="hit._id">
          <v-col>
            <v-row>
              <v-col cols="3" v-html="styleTibetan(getHighlightIfExists(hit, 'definiendum'))">
              </v-col>
              <v-col v-html="styleTibetan(getHighlightIfExists(hit, 'definition'),hit)">
              </v-col>
            </v-row>
            <v-row>
              <div class="source-name">{{getSourceName(hit._index)}}</div>
            </v-row>
          </v-col>
        </v-row>
      </v-col>
    </v-row>
    <v-row v-if="searchHits && searchHits.length > 0 && searchHits.length < numHits">
      <v-col>
        <v-btn @click="doSearch(true)">More</v-btn>
      </v-col>
    </v-row>
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
.search-results-col em, .search-results-col .tibetan em {
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

.source-name {
  text-align: right;
  width: 100%;
  font-size: 60%;
  color: gray;
}

.circle {
  border-radius: 50%;
  padding: 5px 10px;
  margin-right: 5px;
  background: none;
  border: 1px solid #000;
  color: #000;
  text-align: center;
  font: 10px Arial, sans-serif;
}

</style>
