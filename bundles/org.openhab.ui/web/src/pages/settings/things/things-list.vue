<template>
  <f7-page @page:afterin="onPageAfterIn" @page:beforeout="onPageBeforeOut">
    <f7-navbar title="Things" back-link="Settings" back-link-url="/settings/" back-link-force>
      <f7-nav-right>
        <developer-dock-icon />
        <f7-link icon-md="material:done_all" @click="toggleCheck()"
                 :text="(!$theme.md) ? ((showCheckboxes) ? 'Done' : 'Select') : ''" />
      </f7-nav-right>
      <f7-subnavbar :inner="false" v-show="initSeachbar">
        <f7-searchbar
          v-if="initSeachbar"
          ref="searchbar"
          class="searchbar-things"
          custom-search
          @searchbar:search="search"
          @searchbar:clear="clearSearch"
          @searchbar:disable="clearSearch"
          :placeholder="searchPlaceholder"
          :disable-button="!$theme.aurora" />
      </f7-subnavbar>
    </f7-navbar>
    <f7-toolbar class="contextual-toolbar" :class="{ 'navbar': $theme.md }" v-if="showCheckboxes" bottom-ios bottom-aurora>
      <div class="display-flex justify-content-center" v-if="!$theme.md && selectedItems.length > 0" style="width: 100%">
        <f7-link color="red" v-show="selectedItems.length" class="delete display-flex flex-direction-row margin-right" icon-ios="f7:trash" icon-aurora="f7:trash" @click="removeSelected">
          Remove
        </f7-link>
        <f7-link color="orange" v-show="selectedItems.length" class="disable display-flex flex-direction-row margin-right" @click="doDisableEnableSelected(false)" icon-ios="f7:pause_circle" icon-aurora="f7:pause_circle">
          &nbsp;Disable
        </f7-link>
        <f7-link color="green" v-show="selectedItems.length" class="enable display-flex flex-direction-row margin-right" @click="doDisableEnableSelected(true)" icon-ios="f7:play_circle" icon-aurora="f7:play_circle">
          &nbsp;Enable
        </f7-link>
        <f7-link color="blue" v-show="selectedItems.length" class="copy display-flex flex-direction-row" @click="copyFileDefinitionToClipboard(ObjectType.THING, selectedItems)" icon-ios="f7:square_on_square" icon-aurora="f7:square_on_square">
          &nbsp;Copy
        </f7-link>
      </div>
      <f7-link v-if="$theme.md" icon-md="material:close" icon-color="white" @click="showCheckboxes = false" />
      <div class="title" v-if="$theme.md">
        {{ selectedItems.length }} selected
      </div>
      <div class="right" v-if="$theme.md">
        <f7-link v-show="selectedItems.length" tooltip="Disable selected" icon-md="material:pause_circle_outline" icon-color="white" @click="doDisableEnableSelected(false)" />
        <f7-link v-show="selectedItems.length" tooltip="Enable selected" icon-md="material:play_circle_outline" icon-color="white" @click="doDisableEnableSelected(true)" />
        <f7-link v-show="selectedItems.length" tooltip="Remove selected" icon-md="material:delete" icon-color="white" @click="removeSelected" />
        <f7-link v-show="selectedItems.length" tooltip="Copy selected" icon-md="material:content_copy" icon-color="white" @click="copyFileDefinitionToClipboard(ObjectType.THING, selectedItems)" />
      </div>
    </f7-toolbar>

    <f7-list-index
      v-if="ready"
      ref="listIndex"
      v-show="groupBy === 'alphabetical' && !$device.desktop"
      list-el=".things-list"
      :scroll-list="true"
      :label="true" />

    <f7-block class="block-narrow">
      <f7-col v-show="ready">
        <f7-block-title>
          <span>{{ listTitle }}</span>
          <span v-if="showCheckboxes && filteredThings.length">
            -
            <f7-link @click="selectDeselectAll" :text="allSelected ? 'Deselect all' : 'Select all'" />
          </span>
          <template v-if="groupBy === 'location'">
            <div v-if="!$device.desktop && $f7.width < 1024" style="text-align:right; color:var(--f7-block-text-color); font-weight: normal" class="float-right">
              <f7-checkbox :checked="showNoLocation" @change="toggleShowNoLocation" /> <label @click="toggleShowNoLocation" style="cursor:pointer">Show no location</label>
            </div>
            <div v-else style="text-align:right; color:var(--f7-block-text-color); font-weight: normal" class="float-right">
              <label @click="toggleShowNoLocation" style="cursor:pointer">Show no location</label> <f7-checkbox :checked="showNoLocation" @change="toggleShowNoLocation" />
            </div>
          </template>
        </f7-block-title>
      </f7-col>
      <!-- skeleton for not ready -->
      <f7-col v-if="!ready">
        <f7-block-title>&nbsp;Loading...</f7-block-title>
        <f7-list contacts-list class="col things-list">
          <f7-list-group>
            <f7-list-item
              media-item
              v-for="n in 10"
              :key="n"
              :class="`skeleton-text skeleton-effect-blink`"
              title="Label of the thing"
              subtitle="This contains the thing UID"
              after="status badge" />
          </f7-list-group>
        </f7-list>
      </f7-col>

      <f7-col v-else-if="things.length > 0">
        <div class="padding-left padding-right">
          <f7-segmented strong tag="p">
            <f7-button :active="groupBy === 'alphabetical'" @click="switchGroupOrder('alphabetical')">
              Alphabetical
            </f7-button>
            <f7-button :active="groupBy === 'binding'" @click="switchGroupOrder('binding')">
              By binding
            </f7-button>
            <f7-button :active="groupBy === 'location'" @click="switchGroupOrder('location')">
              By location
            </f7-button>
          </f7-segmented>
        </div>
        <f7-list v-if="filteredThings.length === 0">
          <f7-list-item title="Nothing found" />
        </f7-list>
        <f7-list v-else class="col things-list" :contacts-list="groupBy === 'alphabetical'">
          <f7-list-group v-for="(thingsWithInitial, initial) in indexedThings" :key="initial">
            <f7-list-item v-if="thingsWithInitial.length" :title="initial" group-title />
            <f7-list-item
              v-for="(thing, index) in thingsWithInitial"
              :key="index"
              media-item
              class="thinglist-item"
              :checkbox="showCheckboxes"
              :checked="isChecked(thing.UID)"
              :value="thing.UID"
              @click.ctrl="(e) => ctrlClick(e, thing)"
              @click.meta="(e) => ctrlClick(e, thing)"
              @click.exact="(e) => click(e, thing)"
              link=""
              :title="thing.label || thing.UID">
              <div slot="footer">
                {{ thing.UID }}
                <clipboard-icon :value="thing.UID" tooltip="Copy UID" />
              </div>

              <div slot="subtitle" v-if="thing.location && groupBy !== 'location'">
                {{ thing.location }}
                <f7-icon f7="placemark" color="gray" style="font-size: 16px; width: 16px; height: 16px;" />
              </div>
              <f7-badge slot="after" :color="thingStatusBadgeColor(thing.statusInfo)" :tooltip="thing.statusInfo.description">
                {{ thingStatusBadgeText(thing.statusInfo) }}
              </f7-badge>
              <f7-icon v-if="!thing.editable" slot="after-title" f7="lock_fill" size="1rem" color="gray" />
            </f7-list-item>
          </f7-list-group>
        </f7-list>
      </f7-col>
    </f7-block>

    <f7-block v-if="ready && !things.length" class="block-narrow">
      <empty-state-placeholder icon="lightbulb" title="things.title" text="things.text" />
      <f7-row v-if="$f7.width < 1280" class="display-flex justify-content-center">
        <f7-button large fill color="blue" external :href="`${$store.state.websiteUrl}/link/thing`" target="_blank" v-t="'home.overview.button.documentation'" />
      </f7-row>
    </f7-block>

    <f7-fab position="right-bottom" slot="fixed" color="blue" href="add">
      <f7-icon ios="f7:plus" md="material:add" aurora="f7:plus" />
      <!-- <f7-fab-buttons position="top">
        <f7-fab-button label="Scan and add to Inbox">S</f7-fab-button>
        <f7-fab-button label="Add thing manually">M</f7-fab-button>
      </f7-fab-buttons> -->
    </f7-fab>
    <f7-fab position="center-bottom" :text="`Inbox (${inboxCount})`" slot="fixed" :color="inboxCount > 0 ? 'red' : 'gray'" href="inbox">
      <f7-icon f7="tray" />
    </f7-fab>
  </f7-page>
</template>

<style lang="stylus">
.things-list
  margin-bottom calc(var(--f7-fab-size) + 2 * calc(var(--f7-fab-margin) + var(--f7-safe-area-bottom)))
</style>

<script>
import ThingStatus from '@/components/thing/thing-status-mixin'
import ClipboardIcon from '@/components/util/clipboard-icon.vue'
import FileDefinition from '@/pages/settings/file-definition-mixin'

export default {
  mixins: [ThingStatus, FileDefinition],
  props: ['searchFor'],
  components: {
    'empty-state-placeholder': () => import('@/components/empty-state-placeholder.vue'),
    ClipboardIcon
  },
  data () {
    return {
      ready: false,
      initSeachbar: false,
      loading: false,
      things: [],
      inbox: [],
      searchQuery: null,
      filteredThings: [],
      selectedItems: [],
      showCheckboxes: false,
      groupBy: 'alphabetical',
      showNoLocation: false,
      eventSource: null
    }
  },
  computed: {
    indexedThings () {
      const things = this.filteredThings
      if (this.groupBy === 'alphabetical') {
        return things.reduce((prev, thing, i, things) => {
          const initial = (thing.label || thing.UID).substring(0, 1).toUpperCase()
          if (!prev[initial]) {
            prev[initial] = []
          }
          prev[initial].push(thing)

          return prev
        }, {})
      } else if (this.groupBy === 'binding') {
        const bindingGroups = things.reduce((prev, thing, i, things) => {
          const binding = thing.thingTypeUID.split(':')[0]
          if (!prev[binding]) {
            prev[binding] = []
          }
          prev[binding].push(thing)

          return prev
        }, {})
        return Object.keys(bindingGroups).sort((a, b) => a.localeCompare(b)).reduce((objEntries, key) => {
          objEntries[key] = bindingGroups[key]
          return objEntries
        }, {})
      } else {
        const locationGroups = things.reduce((prev, thing, i, things) => {
          if (!thing.location && !this.showNoLocation) return prev
          const location = thing.location || '- No location -'
          if (!prev[location]) {
            prev[location] = []
          }
          prev[location].push(thing)

          return prev
        }, {})
        return Object.keys(locationGroups).sort((a, b) => a.localeCompare(b)).reduce((objEntries, key) => {
          objEntries[key] = locationGroups[key]
          return objEntries
        }, {})
      }
    },
    thingsCount () {
      let sum = 0
      Object.keys(this.indexedThings).forEach(key => {
        sum = sum + this.indexedThings[key].length
      })
      return sum
    },
    inboxCount () {
      return this.inbox.length
    },
    allSelected () {
      return this.selectedItems.length === this.filteredThings.length
    },
    searchPlaceholder () {
      return window.innerWidth >= 1280 ? 'Search (for advanced search, use the developer sidebar (Shift+Alt+D))' : 'Search'
    },
    listTitle () {
      let title = this.filteredThings.length
      if (this.searchQuery) {
        title += ` of ${this.things.length} Things found`
      } else {
        title += ' Things'
      }
      if (this.selectedItems.length > 0) {
        title += `, ${this.selectedItems.length} selected`
      }
      return title
    }
  },
  methods: {
    onPageAfterIn () {
      this.load()
    },
    onPageBeforeOut () {
      this.stopEventSource()
      this.$f7.data.lastThingsSearchQuery = this.$refs.searchbar?.$el.f7Searchbar.query
    },
    load () {
      if (this.loading) return
      this.loading = true

      if (this.initSeachbar) this.$f7.data.lastThingsSearchQuery = this.$refs.searchbar?.f7Searchbar.query
      this.initSeachbar = false

      if (this.searchFor) {
        this.$refs.searchbar?.f7Searchbar.$inputEl.val(this.searchFor)
      }

      this.$oh.api.get('/rest/things?summary=true').then((data) => {
        this.things = data.sort((a, b) => (a.label || a.UID).localeCompare(b.label || a.UID))
        this.filteredThings = this.things
        this.initSeachbar = true
        this.loading = false
        this.ready = true
        this.$nextTick(() => {
          if (this.$refs.listIndex) this.$refs.listIndex.update()
          if (this.$device.desktop && this.$refs.searchbar) {
            this.$refs.searchbar.f7Searchbar.$inputEl[0].focus()
          }
          this.$refs.searchbar?.f7Searchbar.search(this.searchFor || this.$f7.data.lastThingsSearchQuery || '')
        })
        if (!this.eventSource) this.startEventSource()
      })
      this.loadInbox()
    },
    loadInbox () {
      this.$oh.api.get('/rest/inbox?includeIgnored=false').then((data) => {
        this.inbox = data
      })
    },
    switchGroupOrder (groupBy) {
      this.groupBy = groupBy
      const searchbar = this.$refs.searchbar.$el.f7Searchbar
      const filterQuery = searchbar.query
      this.$nextTick(() => {
        if (filterQuery) {
          searchbar.clear()
          searchbar.search(filterQuery)
        }
        if (groupBy === 'alphabetical') this.$refs.listIndex.update()
      })
    },
    toggleShowNoLocation () {
      this.showNoLocation = !this.showNoLocation
    },
    toggleCheck () {
      this.showCheckboxes = !this.showCheckboxes
    },
    selectDeselectAll () {
      if (this.selectedItems.length === this.filteredThings.length) {
        this.selectedItems = []
      } else {
        this.selectedItems = this.filteredThings.map((t) => t.UID)
      }
    },
    search (searchbar, query, previousQuery) {
      this.searchQuery = query.trim().toLowerCase()
      const searchTerms = this.searchQuery.split(',').map(s => s.trim()).filter(s => s)
      if (!searchTerms.length) {
        this.clearSearch()
        return
      }
      this.filteredThings = this.things.filter((thing) => {
        let haystack = [thing.UID, thing.label, thing.location, this.thingStatusBadgeText(thing.statusInfo)]
          .filter(h => h).join('|').toLowerCase()
        return searchTerms.some(t => haystack.includes(t))
      })
      this.selectedItems = this.selectedItems.filter((i) => this.filteredThings.find((thing) => thing.UID === i))
    },
    clearSearch () {
      this.searchQuery = null
      this.filteredThings = this.things
    },
    isChecked (item) {
      return this.selectedItems.indexOf(item) >= 0
    },
    click (event, item) {
      if (this.showCheckboxes) {
        this.toggleItemCheck(event, item.UID, item)
      } else {
        this.$f7router.navigate(item.UID)
      }
    },
    ctrlClick (event, item) {
      this.toggleItemCheck(event, item.UID, item)
      if (!this.selectedItems.length) this.showCheckboxes = false
    },
    toggleItemCheck (event, item) {
      if (!this.showCheckboxes) this.showCheckboxes = true
      if (this.isChecked(item)) {
        this.selectedItems.splice(this.selectedItems.indexOf(item), 1)
      } else {
        this.selectedItems.push(item)
      }
    },
    removeSelected () {
      const vm = this

      this.$f7.dialog.confirm(
        `Remove ${this.selectedItems.length} selected things?`,
        'Remove Things',
        () => {
          vm.doRemoveSelected()
        }
      )
    },
    doRemoveSelected () {
      if (this.selectedItems.some((i) => this.things.find((thing) => thing.UID === i).editable === false)) {
        this.$f7.dialog.alert('Some of the selected things are not modifiable because they have been provisioned by files')
        return
      }

      let dialog = this.$f7.dialog.progress('Deleting Things...')

      const promises = this.selectedItems.map((i) => this.$oh.api.delete('/rest/things/' + i))
      Promise.all(promises).then((data) => {
        this.$f7.toast.create({
          text: 'Things removed',
          destroyOnClose: true,
          closeTimeout: 2000
        }).open()
        this.selectedItems = []
        dialog.close()
        this.load()
      }).catch((err) => {
        dialog.close()
        this.load()
        console.error(err)
        this.$f7.dialog.alert('An error occurred while deleting: ' + err)
      })
    },
    doDisableEnableSelected (enable) {
      let dialog = this.$f7.dialog.progress('Please Wait...')

      const promises = this.selectedItems.map((i) => this.$oh.api.putPlain('/rest/things/' + i + '/enable', enable.toString()))
      Promise.all(promises).then((data) => {
        this.$f7.toast.create({
          text: (enable) ? 'Things enabled' : 'Things disabled',
          destroyOnClose: true,
          closeTimeout: 2000
        }).open()
        this.selectedItems = []
        dialog.close()
        this.load()
      }).catch((err) => {
        dialog.close()
        this.load()
        console.error(err)
        this.$f7.dialog.alert('An error occurred while enabling/disabling: ' + err)
      })
    },
    startEventSource () {
      this.eventSource = this.$oh.sse.connect('/rest/events?topics=openhab/things/*/added,openhab/things/*/removed,openhab/things/*/updated,openhab/things/*/status,openhab/inbox/*/added,openhab/inbox/*/removed', null, (event) => {
        const topicParts = event.topic.split('/')
        if (topicParts[1] === 'inbox') {
          this.loadInbox()
        } else {
          switch (topicParts[3]) {
            case 'status':
              const updatedThing = this.things.find((t) => t.UID === topicParts[2])
              const newStatus = JSON.parse(event.payload)
              if (updatedThing) {
                if (updatedThing.statusInfo.status !== newStatus.status) updatedThing.statusInfo.status = newStatus.status
                if (updatedThing.statusInfo.statusDetail !== newStatus.statusDetail) updatedThing.statusInfo.statusDetail = newStatus.statusDetail
                if (updatedThing.statusInfo.description !== newStatus.description) updatedThing.statusInfo.description = newStatus.description
              }
              break
            case 'added':
            case 'removed':
            case 'updated':
              this.load()
              break
          }
        }
      })
    },
    stopEventSource () {
      this.$oh.sse.close(this.eventSource)
      this.eventSource = null
    }
  }
}
</script>
