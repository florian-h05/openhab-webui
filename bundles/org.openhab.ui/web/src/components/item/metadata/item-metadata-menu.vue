<template>
  <f7-card>
    <f7-card-content v-if="this.editableNamespaces.length > 0">
      <f7-list>
        <ul>
          <f7-list-item
            v-for="namespace in wellKnownNamespaces" :key="namespace.name"
            :link="'/settings/items/' + item.name + '/metadata/' + namespace.name"
            :title="namespace.label"
            :after="namespace.value || 'Not Set'">
            <f7-icon v-if="!namespace.editable" slot="title" f7="lock_fill" size="1rem" color="gray" />
          </f7-list-item>
        </ul>
        <ul v-if="customNamespaces.length > 0">
          <f7-list-item divider />
          <f7-list-item
            v-for="namespace in customNamespaces" :key="namespace.name"
            :link="'/settings/items/' + item.name + '/metadata/' + namespace.name"
            :title="namespace.label"
            :after="namespace.value || 'Not Set'">
            <f7-icon v-if="!namespace.editable" slot="title" f7="lock_fill" size="1rem" color="gray" />
          </f7-list-item>
        </ul>
      </f7-list>
    </f7-card-content>
    <f7-card-footer>
      <f7-button color="blue" @click="addMetadata">
        Add Metadata
      </f7-button>
    </f7-card-footer>
  </f7-card>
</template>

<script>
import MetadataNamespaces from '@/assets/definitions/metadata/namespaces.js'

export default {
  props: ['item'],
  data () {
    return {
      metadataNamespaces: MetadataNamespaces
    }
  },
  beforeMount () {
    if (this.item.type === 'Group' ? (this.item.groupType && this.item.groupType.indexOf('Number:') < 0) : this.item.type.indexOf('Number:') < 0) this.metadataNamespaces = this.metadataNamespaces.filter(n => n.name !== 'unit')
  },
  computed: {
    editableNamespaces () {
      if (!this.item.metadata) return []
      // TODO: determine somehow if other namespaces are not editable
      // (non-managed MetadataProvider)
      // for now we'll assume they're all editable except "semantics"
      return Object.keys(this.item.metadata)
        .filter((n) => n !== 'semantics')
        .map((n) => {
          return {
            name: n,
            value: this.item.metadata[n].value,
            editable: this.item.metadata[n].editable
          }
        })
    },
    wellKnownNamespaces () {
      return this.editableNamespaces
        .filter((n) => this.metadataNamespaces.some((wk) => wk.name === n.name))
        .map((n) => {
          const wellKnown = this.metadataNamespaces.find((wk) => wk.name === n.name)
          return {
            ...n,
            label: wellKnown.label
          }
        })
    },
    customNamespaces () {
      return this.editableNamespaces
        .filter((n) => !this.metadataNamespaces.some((wk) => wk.name === n.name))
        .map((n) => {
          return {
            ...n,
            label: n.name
          }
        })
    }
  },
  methods: {
    editCustomMetadata () {
      this.$f7.dialog.prompt('Please type in the namespace you would like to edit:',
        'Edit Custom Metadata',
        (namespace) => {
          if (namespace) this.$f7.views.main.router.navigate('/settings/items/' + this.item.name + '/metadata/' + namespace)
        })
    },
    addMetadata () {
      this.$f7.actions.create({
        buttons: [
          [
            { label: true, text: 'Well-known namespaces' },
            ...this.metadataNamespaces.map((n) => {
              return {
                text: n.label,
                color: 'blue',
                onClick: () => {
                  this.$f7router.navigate('/settings/items/' + this.item.name + '/metadata/' + n.name)
                }
              }
            })
          ],
          [
            { label: true, text: 'Custom namespaces' },
            { color: 'blue', text: 'Enter Custom Namespace...', onClick: this.editCustomMetadata }
          ],
          [
            { color: 'red', text: 'Cancel', close: true }
          ]
        ]
      }).open()
    }
  }
}
</script>
