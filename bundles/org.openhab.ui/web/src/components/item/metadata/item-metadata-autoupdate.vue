<template>
  <div>
    <f7-list>
      <f7-list-item title="Force auto-update" checkbox :checked="typeof (metadata.value) === 'string' ? metadata.value === 'true' : metadata.value"
                    :indeterminate="metadata.value !== 'true' && metadata.value !== 'false'" :disabled="!editable"
                    @change="(ev) => metadata.value = new Boolean(ev.target.checked).toString()" />
    </f7-list>
    <f7-block-footer class="param-description">
      <small>Force the state to auto-update on command.</small>
    </f7-block-footer>
  </div>
</template>

<script>
import ItemMetadataMixin from '@/components/item/metadata/item-metadata-mixin'

export default {
  props: ['itemName', 'metadata'],
  mixins: [ItemMetadataMixin]
}
</script>
