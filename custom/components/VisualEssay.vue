<template>

  <div v-if="centuryPage">

    <div id="essay-component" ref="" v-html="processedHtml"></div>

    <!-- Entity infobox popup -->
    <div style="display:none;">
      <div ref="popup" class="popup" v-if="hoverEntity">
        <div v-if="hoverEntity.thumbnails" class="image"><img :src="hoverEntity.thumbnails[0]"></div>
        <div class="label" v-html="hoverEntity.label"></div>
        <div v-if="hoverEntity.description" class="description" v-html="hoverEntity.description"></div>
        <div v-if="hoverEntity.summary" class="summary" v-html="hoverEntity.summary"></div>
      </div>
    </div>

  </div>

</template>

<script>

module.exports = {  
  name: 'Essay',
  props: {
    html: { type: String, default: '' },
    path: String,
    anchor: String,
    entities: { type: Object, default: () => ({}) },
    params: { type: Array, default: () => ([]) },
    availableViewers: { type: Array, default: () => ([]) },
    scrollTop: { type: Number, default: 0 },
    contentSource:  { type: Object, default: () => ({}) }
  },
  data: () => ({
    processedHtml: '',
    hoverEntity: null,
    active: null,
    centuryPage: 'false'
  }),
  computed: {
    items() { return this.active ? this.paramsInScope(document.querySelector(`[data-id="${this.active}"] p`)) : [] },
  },
  mounted() {
    document.getElementById('app').classList.add('visual-essay')
  },
  methods: {

    async essayLoaded() {
      this.active = null
      let essayElem = document.getElementById('essay')
      essayElem.querySelectorAll('.seg-link').forEach(el => el.addEventListener('click', () => navigator.clipboard.writeText(el.dataset.anchor)))
      Array.from(essayElem.querySelectorAll('.collapsible')).forEach(el =>el.addEventListener('click', this.toggleExpandCollapse))

      this.tagEntities(essayElem)
      this.$emit('set-entities', this.entities)
      this.$nextTick(() => {
        this.addPopups(this.entities)
        this.convertLinks(essayElem)
        let segments = [...essayElem.querySelectorAll('.segment')]
        if (this.anchor) {
          let anchorElem = document.getElementById(this.anchor)
          if (anchorElem) {
            this.$emit('scroll-to-anchor')
            this.$nextTick(() => essayElem.scrollTop = this.scrollTop - 100)
          }
        } else {
          essayElem.scrollTop = 0
          this.active = segments.length > 0 ? segments[0].dataset.id : null
        }
      })
    },

    doCustomFormatting(elem) {
      Array.from(elem.querySelectorAll('section.cards')).forEach(cardsSection => {
        Array.from(cardsSection.querySelectorAll('section')).forEach(card => {
          ['img', '.card > .segment > p > a', '.card > .segment > p > strong', 'ul'].forEach(selector => {
            let el = card.querySelector(selector)
            if (el) {
              if (selector !== 'strong' || el.parentElement.tagName !== 'A') card.appendChild(el)
            }
          })
          let segments = []
          Array.from(card.querySelectorAll('.segment')).forEach(seg => {
            if (seg.textContent.trim() === '') {
              card.removeChild(seg)
            } else {
              segments.push(seg)
            }
          })
          if (segments.length > 0) {
            let abstractWrapper = document.createElement('div')
            abstractWrapper.classList.add('abstract')
            card.appendChild(abstractWrapper)
            let abstractDiv = document.createElement('div')
            abstractDiv.classList.add('abstract-text')
            abstractWrapper.appendChild(abstractDiv)
            segments.forEach(seg => abstractDiv.appendChild(seg))
            abstractWrapper.innerHTML += '<button class="collapsible">read more</button>'
          }
        })
      })
      return elem.innerHTML
    },

    // Adds tippy popups to tagged entity text
    addPopups(entities) {
      tippy('.entity', {
        allowHTML: true,
        interactive: true,
        appendTo: document.body,
        delay: [null, null],
        placement: 'right',
        theme: 'light-border',
        onShow: async (instance) => {
          this.hoverEntity = this.entities[instance.reference.dataset.eid]
          if (!this.hoverEntity.summary) {
            let label, summary
            if (this.hoverEntity.article) {
              let resp = await fetch(this.hoverEntity.article)
              let articleMarkdown = await resp.text()
              let tmp = new DOMParser().parseFromString(md.render(articleMarkdown), 'text/html').children[0].children[1]
              label = tmp.querySelector('h1, h2, h3, h4, h5, h6').innerHTML
              summary = Array.from(tmp.querySelectorAll('p')).map(p => p.outerHTML).join('')
            } else if (this.hoverEntity.mwPage) {
              let page = this.hoverEntity.mwPage.replace(/\/w\//, '/wiki/').split('/wiki/').pop()
              let resp = await fetch(`https://en.wikipedia.org/api/rest_v1/page/summary/${page}`)
              resp = await resp.json()
              summary = resp.extract_html
            }
            if (label || summary) {
              entities[instance.reference.dataset.eid].summary = summary
              entities[instance.reference.dataset.eid].label = entities[instance.reference.dataset.eid].label || label
              this.$emit('set-entities', entities)
              this.hoverEntity = {...entities[instance.reference.dataset.eid]}
            }
          }
          this.$nextTick(() => instance.setContent(this.$refs.popup.outerHTML))
        },
        onHide: () => {}
      })
    },

    toggleExpandCollapse(e) {
 
    },

    // Finds all param tags in elements between top-level app element and element in para arg
    paramsInScope(root) {
      
    }

  },
  watch: {

    html: {
      handler: function (html) {
        if (html) {
          let tmp = new DOMParser().parseFromString(html, 'text/html').children[0].children[1]
          this.processedHtml = this.doCustomFormatting(tmp)
          this.$nextTick(() => this.essayLoaded())
        }
      },
      immediate: true
    },

    active: {
      handler: function (current, prior) {
        this.$emit('set-active', current)
        let activeSegment = document.querySelector(`[data-id="${current}"]`)
        if (activeSegment) activeSegment.classList.add('active')
        let priorSegment = document.querySelector(`[data-id="${prior}"]`)
        if (priorSegment) priorSegment.classList.remove('active')
      },
      immediate: true
    },

    items: {
      handler: function (items) {
        this.$emit('set-items', items)
      },
      immediate: true
    },

    scrollTop: {
      handler: function (pos) {
        let target = this.$refs.essay
        if (pos && target) {
          let segments = Array.from(target.querySelectorAll('.segment'))
          let i
          for (i = 0; i < segments.length; i++) {
            if (pos <= segments[i].offsetTop + segments[i].clientHeight - 200) break
          }
          if (i < segments.length && this.active !== segments[i].dataset.id ) this.active = segments[i].dataset.id
        }
      },
      immediate: true
    }

  }
}

</script>

<style>
  #essay-component {
    padding-left: 3%;
    padding-right: 3%;
  }
</style>