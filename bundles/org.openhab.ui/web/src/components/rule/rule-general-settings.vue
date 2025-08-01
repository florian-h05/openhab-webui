<template>
  <div>
    <f7-block v-if="ready" class="block-narrow">
      <f7-col>
        <f7-list inline-labels no-hairlines-md>
          <f7-list-input ref="ruleId" :label="`${type} ID`" type="text" :placeholder="`A unique identifier for the ${type.toLowerCase()}`" :value="rule.uid" required :validate="editable"
                         :disabled="!createMode" :info="(createMode) ? 'Required. Note: cannot be changed after the creation' : ''" input-id="input"
                         pattern="[A-Za-z0-9_\-]+" error-message="Required. A-Z,a-z,0-9,_,- only"
                         @input="rule.uid = $event.target.value" :clear-button="createMode">
            <f7-link slot="inner" icon-f7="hammer_fill" style="margin-top: 4px; margin-left: 4px; margin-bottom: auto" tooltip="Fix ID" v-if="createMode && $refs.ruleId?.state?.inputInvalid && rule.uid.trim()" @click="$oh.utils.normalizeInput('#input')" />
          </f7-list-input>
          <f7-list-input v-if="!createMode && templateName" label="Template" type="text" :value="templateName" disabled />
          <f7-list-input label="Label" type="text" :placeholder="`${type} label for display purposes`" :info="(createMode) ? 'Required' : ''" :value="rule.name" required validate
                         :disabled="!editable" @input="rule.name = $event.target.value" :clear-button="editable" />
          <f7-list-input label="Description" type="text" :value="rule.description"
                         :disabled="!editable" @input="rule.description = $event.target.value" :clear-button="editable" />
        </f7-list>
        <f7-list inline-labels no-hairlines-md>
          <tag-input v-if="!stubMode" title="Tags" :item="rule" :disabled="!editable" :showSemanticTags="true" :inScriptEditor="inScriptEditor" :inSceneEditor="inSceneEditor" />
        </f7-list>
      </f7-col>
    </f7-block>

    <!-- skeletons for not ready-->
    <f7-block v-else class="block-narrow">
      <f7-col class="skeleton-text skeleton-effect-blink">
        <f7-list inline-labels no-hairlines-md>
          <f7-list-input label="Rule ID" type="text" placeholder="Required" value="_______" required :validate="editable"
                         :disabled="true" :info="(createMode) ? 'Note: cannot be changed after the creation' : ''"
                         @input="rule.uid = $event.target.value" :clear-button="createMode" />
          <f7-list-input label="Name" type="text" placeholder="Required" required validate
                         :disabled="true" @input="rule.name = $event.target.value" :clear-button="editable" />
          <f7-list-input label="Description" type="text" value="__ _____ ___ __ ___"
                         :disabled="true" @input="rule.description = $event.target.value" :clear-button="editable" />
        </f7-list>
        <f7-list inline-labels no-hairlines-md>
          <tag-input v-if="!stubMode" :item="rule" :disabled="!editable" :showSemanticTags="true" :inScriptEditor="inScriptEditor" :inSceneEditor="inSceneEditor" />
        </f7-list>
      </f7-col>
    </f7-block>
  </div>
</template>

<script>
import TagInput from '@/components/tags/tag-input.vue'

export default {
  props: ['rule', 'ready', 'createMode', 'stubMode', 'templateName', 'inScriptEditor', 'inSceneEditor'],
  components: {
    TagInput
  },
  computed: {
    editable () {
      return this.createMode || (this.rule && this.rule.editable)
    },
    type () {
      if (this.inScriptEditor) return 'Script'
      if (this.inSceneEditor) return 'Scene'
      return 'Rule'
    }
  },
  methods: {
    isScriptTag (tag) {
      if (this.inScriptEditor !== true) return false
      if (tag === 'Script') return true
    },
    isSceneTag (tag) {
      if (this.inSceneEditor !== true) return false
      if (tag === 'Scene') return true
    }
  }
}
</script>
