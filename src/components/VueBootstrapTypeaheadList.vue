<template>
  <div class="list-group shadow" ref="suggestionList">
    <vue-bootstrap-typeahead-list-item
      v-for="(item, id) in matchedItems" :key="id"
      :active="isListItemActive(id)"
      :data="item.data"
      :html-text="highlight(item.text)"
      :background-variant="backgroundVariant"
      :text-variant="textVariant"
      @click.native="handleHit(item, $event)"
    >
      <template v-if="$scopedSlots.suggestion" slot="suggestion" slot-scope="{ data, htmlText }">
        <slot name="suggestion" v-bind="{ data, htmlText }" />
      </template>
    </vue-bootstrap-typeahead-list-item>
  </div>
</template>

<script>
import VueBootstrapTypeaheadListItem from './VueBootstrapTypeaheadListItem.vue'

function sanitize(text) {
  return text ? text.replace(/</g, '&lt;').replace(/>/g, '&gt;') : null
}

function escapeRegExp(str) {
  return str ? str.replace(/[.*+?^${}()|[\]\\]/g, '\\$&') : null
}

export default {
  name: 'VueBootstrapTypeaheadList',

  components: {
    VueBootstrapTypeaheadListItem
  },

  props: {
    data: {
      type: Array,
      required: true,
      validator: d => d instanceof Array
    },
    query: {
      type: String,
      default: null
    },
    backgroundVariant: {
      type: String
    },
    textVariant: {
      type: String
    },
    maxMatches: {
      type: Number,
      default: 10
    },
    minMatchingChars: {
      type: Number,
      default: 2
    }
  },

  data() {
    return {
      activeListItem: -1
    }
  },

  computed: {
    highlight() {
      return (text) => {
        text = sanitize(text)
        if (this.query && this.query.length === 0) {
          return text
        }
        const re = new RegExp(this.escapedQuery, 'gi')

        return text ? text.replace(re, `<span class="vbt-highlighted">$&</span>`) : null
      }
    },

    escapedQuery() {
      return escapeRegExp(sanitize(this.query))
    },

    matchedItems() {
      if (this.query && (this.query.length === 0 || this.query.length < this.minMatchingChars)) {
        return []
      }

      const re = new RegExp(this.escapedQuery, 'gi')

      // Filter, sort, and concat
      return this.data
        .filter(i => i.text.match ? i.text.match(re) !== null : false)
        .sort((a, b) => {
          const aIndex = a.text.indexOf(a.text.match(re)[0])
          const bIndex = b.text.indexOf(b.text.match(re)[0])

          if (aIndex < bIndex) { return -1 }
          if (aIndex > bIndex) { return 1 }
          return 0
        }).slice(0, this.maxMatches)
    }
  },

  created () {
    this.$parent.$on('input', this.resetActiveListItem)
    this.$parent.$on('keyup.down', this.selectNextListItem)
    this.$parent.$on('keyup.up', this.selectPreviousListItem)
    this.$parent.$on('keyup.enter', this.hitActiveListItem)
  },

  watch: {
    activeListItem (newValue, oldValue) {
      if (newValue >= 0) {
        const scrollContainer = this.$refs.suggestionList
        const listItem = scrollContainer.children[this.activeListItem]
        const scrollContainerlHeight = scrollContainer.clientHeight
        const listItemHeight = listItem.clientHeight
        const visibleItems = Math.floor(
          scrollContainerlHeight / listItemHeight
        )
        if (newValue >= visibleItems) {
          scrollContainer.scrollTop = listItemHeight * this.activeListItem
        } else {
          scrollContainer.scrollTop = 0
        }
      }
    }
  },

  methods: {
    handleHit(item, evt) {
      this.$emit('hit', item)
      evt.preventDefault()
    },
    isListItemActive(id) {
      return this.activeListItem === id
    },
    resetActiveListItem() {
      this.activeListItem = -1
    },
    selectNextListItem() {
      if (this.activeListItem < this.matchedItems.length - 1) {
        this.activeListItem++
      } else {
        this.activeListItem = -1
      }
    },
    selectPreviousListItem() {
      if (this.activeListItem < 0) {
        this.activeListItem = this.matchedItems.length - 1
      } else {
        this.activeListItem--
      }
    },
    hitActiveListItem() {
      if (this.activeListItem >= 0) {
        this.$emit('hit', this.matchedItems[this.activeListItem])
        this.activeListItem--
      }
    }
  }
}
</script>

<style scoped>
  .vbt-highlighted {
    font-weight: bold;
  }
</style>
