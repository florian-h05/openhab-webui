<template>
  <f7-popup ref="modulePopup" class="moduleconfig-popup">
    <f7-page>
      <f7-navbar>
        <f7-nav-left>
          <f7-link icon-ios="f7:arrow_left" icon-md="material:arrow_back" icon-aurora="f7:arrow_left" popup-close />
        </f7-nav-left>
        <f7-nav-title>
          {{ action.label }}
        </f7-nav-title>
        <f7-nav-right>
          <f7-link @click="close">
            Close
          </f7-link>
        </f7-nav-right>
      </f7-navbar>
      <f7-block class="no-margin no-padding">
        <f7-block-footer class="margin-horizontal">
          {{ action.description }}
        </f7-block-footer>
        <!-- Action Inputs -->
        <f7-col>
          <f7-block-title class="parameter-group-title">
            Action Input
          </f7-block-title>
          <config-sheet v-if="inputConfigDescriptions.length > 0" ref="configSheet"
                        :parameter-groups="[]" :parameters="inputConfigDescriptions"
                        :configuration="actionInput" :read-only="executing" />
          <div class="margin" v-else>
            There is no input to be configured for this action.
          </div>
        </f7-col>
        <!-- Executing Spinner -->
        <f7-col v-if="executing" class="text-align-center padding-top margin-top">
          <f7-block-title>
            <f7-preloader :size="30" />
            <div>Executing...</div>
          </f7-block-title>
        </f7-col>
        <!-- Execute Button -->
        <f7-col v-if="!executing">
          <f7-list>
            <f7-list-button color="blue" title="Execute Action" @click="execute" />
          </f7-list>
        </f7-col>
        <!-- Action Outputs -->
        <f7-col v-if="!executing && actionOutput">
          <f7-block-title class="parameter-group-title">
            Action Output
          </f7-block-title>
          <div v-if="Object.keys(actionOutput).length === 0" class="margin">
            There is either no output for this action or something went wrong - please check the logs.
          </div>
          <div v-else>
            <f7-list media-list>
              <template v-for="key of Object.keys(actionOutput)">
                <!-- Render result as a list item, works without action output definition from REST -->
                <f7-list-item v-if="key === 'result'" :key="key" :floating-label="$theme.md" :title="action.outputs.find(o => o.name === key)?.label || 'Result'" :footer="action.outputs.find(o => o.name === key)?.description">
                  <div slot="after">
                    {{ actionOutput[key] }}
                  </div>
                </f7-list-item>
                <!-- Render QR code if the key is qrCode -->
                <!-- Render QR code if the action output type is qrCode in the action output definition from REST -->
                <f7-list-item v-else-if="key === 'qrCode' || action.outputs.find(o => o.name === key)?.type === 'qrCode'" :key="key" :floating-label="$theme.md" :title="action.outputs.find(o => o.name === key)?.label || 'QR Code'" :footer="action.outputs.find(o => o.name === key)?.description">
                  <vue-qrcode :value="actionOutput[key]" slot="after" />
                </f7-list-item>
                <!-- Render other keys as list items with the label defined by the action output definition from REST or the key as label -->
                <f7-list-item v-else :key="key" :floating-label="$theme.md" :title="action.outputs.find(o => o.name === key)?.label || key" :footer="action.outputs.find(o => o.name === key)?.description">
                  <div slot="after">
                    {{ actionOutput[key] }}
                  </div>
                </f7-list-item>
              </template>
              <f7-list-item accordion-item title="Raw Output Value">
                <f7-accordion-content class="thing-type-description">
                  <div class="margin">
                    <code> {{ actionOutput }} </code>
                  </div>
                </f7-accordion-content>
              </f7-list-item>
            </f7-list>
          </div>
        </f7-col>
      </f7-block>
    </f7-page>
  </f7-popup>
</template>

<script>
import ConfigSheet from '@/components/config/config-sheet.vue'

export default {
  components: {
    ConfigSheet,
    'vue-qrcode': () => import(/* webpackChunkName: "vue-qrcode" */ 'vue-qrcode')
  },
  props: ['thingUID', 'action'],
  data () {
    return {
      executing: false,
      actionInput: {},
      actionOutput: null
    }
  },
  computed: {
    inputConfigDescriptions () {
      return this.action.inputConfigDescriptions.map((c) => {
        if (!c.label) c.label = c.name
        return c
      })
    }
  },
  methods: {
    execute () {
      if (this.$refs.configSheet?.isValid() === false) {
        this.$f7.dialog.alert('Please review the input and correct validation errors')
        return
      }
      this.executing = true
      this.$oh.api.post(`/rest/actions/${this.thingUID}/${encodeURIComponent(this.action.actionUid)}`, this.actionInput)
        .then((data) => {
          this.actionOutput = data
          this.executing = false
        })
    },
    close () {
      this.$refs.modulePopup.close()
    }
  }
}
</script>
