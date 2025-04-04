<template>
  <f7-col>
    <f7-list inline-labels accordion-list no-hairline-md class="no-margin-top">
      <f7-list-input ref="pageId" label="Page ID" type="text" placeholder="A unique identifier for the page" :value="page.uid" @input="page.uid = $event.target.value"
                     :clear-button="createMode" :info="(createMode) ? 'Required. Note: cannot be changed after the creation' : ''" input-id="input"
                     required validate pattern="[A-Za-z0-9_]+" error-message="Required. A-Z,a-z,0-9,_ only" :disabled="!createMode">
        <f7-link slot="inner" icon-f7="hammer_fill" style="margin-top: 4px; margin-left: 4px; margin-bottom: auto" tooltip="Fix ID" v-if="createMode && $refs.pageId?.state?.inputInvalid && page.uid.trim()" @click="$oh.utils.normalizeInput('#input')" />
      </f7-list-input>
      <f7-list-input label="Label" type="text" placeholder="Page label used for display purposes" :info="(createMode) ? 'Required' : ''" :value="page.config.label" @input="page.config.label = $event.target.value" required validate clear-button />
      <f7-list-item accordion-item title="Sidebar &amp; Visibility" v-if="page.uid !== 'overview'">
        <f7-accordion-content>
          <f7-list-item ref="pageVisibility" title="Visible only to" smart-select :smart-select-params="{openIn: 'popover'}">
            <select name="pagevisibility" multiple @change="updatePageVisibility">
              <optgroup label="Roles">
                <option value="role:administrator" :selected="isVisibleTo('role:administrator')">
                  Administrators
                </option>
                <option value="role:user" :selected="isVisibleTo('role:user')">
                  Users
                </option>
              </optgroup>
            </select>
          </f7-list-item>
          <f7-list inline-labels no-hairline-md>
            <f7-list-item title="Show on sidebar">
              <f7-toggle slot="after" :checked="page.config.sidebar" @toggle:change="page.config.sidebar = $event" />
            </f7-list-item>
            <f7-list-input label="Sidebar order" type="number" placeholder="Assign order index to rearrange pages on sidebar" :value="page.config.order" @input="page.config.order = $event.target.value" clear-button />
            <f7-list-input label="Icon" type="text" placeholder="Assign a custom icon" :value="page.config.icon" @input="page.config.icon = $event.target.value" clear-button />
            <f7-list-input label="Browser Title" type="text" placeholder="A custom browser title instead of the label" :value="page.config.browserTitle" @input="page.config.browserTitle = $event.target.value" clear-button />
          </f7-list>
        </f7-accordion-content>
      </f7-list-item>
    </f7-list>
    <template v-if="page.uid !== 'overview'">
      <f7-list inline-labels no-hairline-md>
        <tag-input :item="page" />
      </f7-list>
      <f7-list v-if="!createMode" inline-labels no-hairline-md>
        <f7-list-button color="blue" @click="duplicatePage">
          Duplicate Page
        </f7-list-button>
        <f7-list-button color="red" @click="deletePage">
          Remove Page
        </f7-list-button>
      </f7-list>
    </template>
  </f7-col>
</template>

<script>
import TagInput from '@/components/tags/tag-input.vue'
import cloneDeep from 'lodash/cloneDeep'

export default {
  components: {
    TagInput
  },
  props: ['page', 'createMode'],
  data () {
    return {}
  },
  methods: {
    isVisibleTo (userrole) {
      return Array.isArray(this.page.config.visibleTo) && this.page.config.visibleTo.indexOf(userrole) >= 0
    },
    updatePageVisibility (userrole) {
      let value = this.$refs.pageVisibility.f7SmartSelect.getValue()
      if (value && value.length === 0) {
        this.$delete(this.page.config, 'visibleTo')
      } else {
        this.$set(this.page.config, 'visibleTo', value)
        this.$f7.toast.create({
          text: 'Please be advised: the visibility restriction is not a security feature - items can be controlled by other means!',
          closeButton: true,
          destroyOnClose: true
        }).open()
      }
    },
    duplicatePage () {
      const pageClone = cloneDeep(this.page)
      const pageType = pageClone.component.replace(/^oh-|-page$/g, '')
      pageClone.uid = pageClone.uid + '_copy'
      this.$f7router.navigate(`/settings/pages/${pageType}/add`, { props: { createMode: true, pageCopy: pageClone } })
    },
    deletePage () {
      this.$f7.dialog.confirm(
        `Are you sure you want to delete ${this.page.uid}?`,
        'Delete Page',
        () => {
          this.$oh.api.delete('/rest/ui/components/ui:page/' + this.page.uid).then((data) => {
            this.$f7.toast.create({
              text: `Page '${this.page.uid}' deleted`,
              destroyOnClose: true,
              closeTimeout: 2000
            }).open()
            this.$f7router.back('/settings/pages/', { force: true })
          }).catch((err) => {
            console.error(err)
            this.$f7.dialog.alert('An error occurred while deleting: ' + err)
          })
        }
      )
    }
  }
}
</script>
