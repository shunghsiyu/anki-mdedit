<template>
  <div id="app">
    <form v-cloak v-on:keydown.meta.s.prevent="downloadCsv" v-on:keydown.meta.enter.prevent="newCard">
    <my-input v-for="(card, index) in cards" key="card.frontText+index" @duplicate="duplicateCard" @remove="removeCard" @next="activateNextCard" @previous="activatePreviousCard" :card="card" :index="index" :do-then-persist="doThenPersist"></my-input>
    <p class="card-num">There are currently <span v-text="cards.length"></span> cards.</p>
    <button class="mdl-button mdl-js-button mdl-button--raised mdl-js-ripple-effect mdl-button--accent" type="submit" v-on:click.prevent="newCard">Add New Card</button>
    <button class="mdl-button mdl-js-button mdl-button--raised mdl-js-ripple-effect mdl-button--colored" type="submit" v-on:click.prevent="downloadCsv">Download CSV File</button>
    <button class="mdl-button mdl-js-button mdl-button--raised mdl-js-ripple-effect mdl-button--colored" type="submit" v-on:click.prevent="restore" v-bind:disabled="!oldCards">Undelete Card</button>
    <button class="mdl-button mdl-js-button mdl-button--raised mdl-js-ripple-effect mdl-button--colored" type="submit" v-on:click.prevent="reset">Reset</button>
  </div>
</template>

<script>
import Vue from 'vue'
import _ from 'lodash'
import showdown from 'showdown'
import Papa from 'papaparse'
import { saveAs } from 'file-saver'

var Blob = window.Blob
var localStorage = window.localStorage
var Event = window.Event
var componentHandler = require('exports?componentHandler!material-design-lite/material.js')

var publicDocUri = function () {
  return [{
    type: 'lang',
    regex: /(docs\.google\.com\/document\/d\/[\w-]+)\/edit#bookmark=/g,
    replace: '$1/pub#'
  }, {
    type: 'output',
    regex: /<img src="(data:([^;]+);[^"]+)" alt="([^"]*)" \/>/g,
    replace: '<object type="$2" data="$1">$3</object>'
  }]
}

var Singleton = (function () {
  var instance = null

  function createInstance () {
    var object = new showdown.Converter({
      extensions: [publicDocUri]
    })
    object.setOption('strikethrough', true)
    object.setOption('headerLevelStart', 3)
    object.setOption('noHeaderId', true)
    object.setOption('tables', true)
    return object
  }

  return {
    getInstance: function () {
      if (!instance) {
        instance = createInstance()
      }
      return instance
    }
  }
})()

Vue.config.keyCodes = {
  d: 68,
  s: 83
}

Vue.component('my-input', {
  props: ['card', 'index', 'doThenPersist'],
  template: '<div class="card mdl-card mdl-shadow--4dp"><div class="remove mdl-card__menu"><button class="remove mdl-button mdl-js-button mdl-button--icon" type="submit" v-on:click.prevent="removeThis"><i class="material-icons">delete</i></button></div><div class="mdl-card__title mdl-card--border"><div class="front-view" v-html="card.front"></div></div><div class="mdl-card__supporting-text mdl-card--border"><div class="back-view" v-html="card.back"></div></div><div class="input-area mdl-card__actions mdl-card--border" @keydown.meta.d.prevent="duplicateEvent" @keydown.alt.tab.prevent="previousField" @keydown.alt.down="nextEvent" @keydown.alt.up="previousEvent"><div class="mdl-textfield mdl-js-textfield"><textarea ref="frontInput" :id="\'front-input\'+index" class="front mdl-textfield__input" :rows="frontTextRows" v-model="card.frontText"></textarea><label class="mdl-textfield__label" :for="\'front-input\'+index">Modify content on the front here...</label></div><div class="mdl-textfield mdl-js-textfield"><textarea  ref="backInput" :id="\'back-input\'+index" class="back mdl-textfield__input" :rows="backTextRows" v-model="card.backText"></textarea><label class="mdl-textfield__label" for="\'back-input\'+index">Modify content on the back here...</label></div><div class="mdl-textfield mdl-js-textfield mdl-textfield--floating-label"><input ref="urlInput" :id="\'url-input\'+index" class="url-input mdl-textfield__input" type="text" v-model="card.url"><label class="mdl-textfield__label" :for="\'url-input\'+index">URL of Note</label></div></div></div>',
  created () {
    this.renderFront()
    this.renderBack()
  },
  mounted () {
    componentHandler.upgradeElements(this.$el)
  },
  updated () {
    var fakeInput = new Event('input')
    /* Hack for MDL */
    _.map(this.$refs, ref => ref.dispatchEvent(fakeInput))
  },
  beforeDestroy () {
    componentHandler.downgradeElements(this.$el)
  },
  data () {
    return {
      debounce: 100,
      timer: [0, 0, 0]
    }
  },
  watch: {
    'card.frontText': function () {
      clearTimeout(this.timer[0])
      this.timer[0] = setTimeout(() => this.doThenPersist(this.renderFront), this.debounce)
    },
    'card.backText': function () {
      clearTimeout(this.timer[1])
      this.timer[1] = setTimeout(() => this.doThenPersist(this.renderBack), this.debounce)
    },
    'card.url': function () {
      clearTimeout(this.timer[2])
      this.timer[2] = setTimeout(() => this.doThenPersist(this.renderBack), this.debounce)
    }
  },
  methods: {
    activateCard () {
      var frontTextElement = this.$refs.frontInput
      this.activateElement(frontTextElement)
    },
    activateElement (el) {
      el.focus()
      el.select()
    },
    nextEvent () {
      this.$emit('next', this)
    },
    previousEvent () {
      this.$emit('previous', this)
    },
    previousField () {
      var currentElement = document.activeElement
      var refs = this.$refs
      var lookup = {
        [refs.frontInput]: null,
        [refs.backInput]: refs.frontInput,
        [refs.urlInput]: refs.backInput
      }
      var el = lookup[currentElement]
      if (el) {
        this.activateElement(el)
      }
    },
    duplicateEvent () {
      this.$emit('duplicate', this.card)
    },
    renderFront () {
      var text = this.card.frontText || '### Front of the Card Preview'
      this.card.front = this.renderMarkdown(text)
    },
    renderBack () {
      var text = this.card.backText || '### Back of the Card Preview'
      var extra = this.card.url ? '\n\n[note](' + this.card.url + ')' : ''
      this.card.back = this.renderMarkdown(text + extra)
    },
    renderMarkdown (text) {
      var converter = Singleton.getInstance()
      return converter.makeHtml(text)
    },
    removeThis () {
      this.$emit('remove', this.card)
    },
    numRow (text) {
      return text.split('\n').length + 1
    }
  },
  computed: {
    frontTextRows () {
      return this.numRow(this.card.frontText)
    },
    backTextRows () {
      return this.numRow(this.card.backText)
    }
  }
})

var defaultData = {
  cards: [],
  oldCards: null
}
var data = localStorage.getItem('data') ? JSON.parse(localStorage.getItem('data')) : _.cloneDeep(defaultData)

var emptyCard = {
  front: '',
  back: '',
  frontText: '',
  backText: '',
  url: ''
}

export default {
  name: 'app',
  data () {
    return _.cloneDeep(data)
  },
  methods: {
    newCard () {
      this.addNewCard()
      /* Set focus on the new card */
      setTimeout(() => _.last(this.$children).activateCard())
    },
    addNewCard (card) {
      var cardTemplate = card || emptyCard
      var cardClone = JSON.parse(JSON.stringify(cardTemplate))
      this.cards.push(cardClone)
    },
    duplicateCard (card) {
      this.addNewCard(card)
    },
    displayTsv () {
      var data = this.cards
      var tsvStr = Papa.unparse(data, {
        delimiter: '\t',
        header: false
      })
      console.log(tsvStr)
    },
    downloadCsv () {
      var data = this.cards
      var csvStr = Papa.unparse(data)
      var blob = new Blob([csvStr], {type: 'text/csv'})
      saveAs(blob, 'output.csv')
      this.displayTsv()
    },
    removeCard (card) {
      this.backup()
      var index = this.cards.indexOf(card)
      if (index !== -1) {
        this.cards.splice(index, 1)
      }
      this.persist()
    },
    activatePreviousCard (current) {
      var currentIndex = this.$children.indexOf(current)
      if (currentIndex > 0) {
        this.$children[currentIndex - 1].activateCard()
      }
    },
    activateNextCard (current) {
      var currentIndex = this.$children.indexOf(current)
      if (currentIndex < this.$children.length - 1) {
        this.$children[currentIndex + 1].activateCard()
      }
    },
    persist () {
      localStorage.setItem('data', JSON.stringify(this.$data))
    },
    doThenPersist (func) {
      func()
      this.persist()
    },
    reset () {
      this.backup()
      var data = _.cloneDeep(defaultData)
      this.cards = data.cards
      localStorage.removeItem('data')
    },
    backup () {
      this.oldCards = _.clone(this.cards)
    },
    restore () {
      this.cards = this.oldCards
      this.oldCards = null
    }
  }
}
</script>

<style lang="less">
body {
  background: #EEE;
  box-sizing: border-box;
  padding: 10px;
  margin: 0;
}

#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  color: #2c3e50;
  margin-top: 60px;

  max-width: 600px;
  margin: 0 auto;
  
  button.mdl-button {
    margin: 8px 5px;
  }
  
  p.card-num {
    text-align: center;
  }
}

.card.mdl-card {
  width: 100%;
  margin: 20px auto;
  box-sizing: border-box;
  border-radius: 10px;
  
  .remove.mdl-card__menu {
    top: auto;
    bottom: -10px;
    right: -10px;
    
    i {
      color: rgb(33,150,243);
    }
  }
  
  .front-view {
    padding-top: 10px;
  }
  
  .front-view, .back-view {
    p {
      font-size: 140%;
      font-weight: 300;
    }
    
    blockquote p {
      font-size: 80%;
    }
  }
  
  .mdl-card__title, .mdl-card__supporting-text, .front-view, .back-view, .mdl-textfield {
    box-sizing: border-box;
    width: 100%;
  }
  
  .mdl-card__title.mdl-card--border {
    border-bottom: 1px solid;
    border-image: linear-gradient(to right, rgba(0, 0, 0, 0) 5%,  rgba(0, 0, 0, 0.1) 30%, rgba(0, 0, 0, 0.1) 70%, rgba(0, 0, 0, 0) 95%) 1;
  }

  .mdl-card__actions {
    textarea, input, label {
      box-sizing: border-box;
      padding: 5px 5px;
    }
    
    textarea, input {
      background: rgba(0, 0, 0, 0.03);
      border-radius: 5px;
    }
  }
  
}
</style>
