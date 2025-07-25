<template>
  <f7-page @page:afterin="onPageAfterIn" @page:afterout="onPageAfterOut">
    <f7-navbar :title="(createMode ? (ruleCopy ? 'Duplicate rule' : 'Create rule') : stubMode ? 'Regenerate rule from template' : rule.name) + dirtyIndicator" :subtitle="hasOpaqueModule ? opaqueModulesTypeText : undefined" back-link="Back" no-hairline>
      <f7-nav-right>
        <developer-dock-icon />
        <template v-if="isEditable">
          <f7-link @click="save()" v-if="$theme.md" icon-md="material:save" icon-only />
          <f7-link @click="save()" v-if="!$theme.md">
            {{ stubMode ? $t('dialogs.regenerate') : $t(createMode ? 'dialogs.create' : 'dialogs.save') }} <span v-if="$device.desktop">&nbsp;(Ctrl-S)</span>
          </f7-link>
        </template>
        <f7-link v-else icon-f7="lock_fill" icon-only tooltip="This rule is not editable through the UI" />
      </f7-nav-right>
    </f7-navbar>
    <f7-toolbar tabbar position="top">
      <f7-link @click="switchTab('design', fromYaml)" :tab-link-active="currentTab === 'design'" class="tab-link">
        Design
      </f7-link>
      <f7-link v-if="ready && !(hasSource && hasOpaqueModule)" @click="switchTab('code', toYaml)" :tab-link-active="currentTab === 'code'" class="tab-link">
        Code
      </f7-link>
      <f7-link v-if="ready && hasSource" @click="switchTab('source')" :tab-link-active="currentTab === 'source'" class="tab-link">
        Source
      </f7-link>
    </f7-toolbar>
    <f7-tabs class="sitemap-editor-tabs">
      <f7-tab id="design" @tab:show="() => this.currentTab = 'design'" :tab-active="currentTab === 'design'">
        <f7-block v-if="ready && rule.status && !createMode && !stubMode" class="block-narrow padding-left padding-right" strong>
          <f7-col v-if="!createMode && !stubMode">
            <div class="float-right align-items-flex-start align-items-center">
              <!-- <f7-toggle class="enable-toggle"></f7-toggle> -->
              <f7-link v-if="canRegenerate" :color="$f7.data.themeOptions.dark === 'dark' ? 'purple' : 'deeppurple'" :tooltip="'Regenerate from template'" icon-md="f7:arrow_2_circlepath" icon-ios="f7:arrow_2_circlepath" icon-aurora="f7:arrow_2_circlepath" icon-size="32" @click="regenerateFromTemplate" />
              <f7-link :color="(rule.status.statusDetail === 'DISABLED') ? 'orange' : 'gray'" :tooltip="((rule.status.statusDetail === 'DISABLED') ? 'Enable' : 'Disable') + (($device.desktop) ? ' (Ctrl-D)' : '')" icon-ios="f7:pause_circle" icon-md="f7:pause_circle" icon-aurora="f7:pause_circle" icon-size="32" @click="toggleDisabled" />
              <f7-link :tooltip="'Run Now' + (($device.desktop) ? ' (Ctrl-R)' : '')" icon-ios="f7:play_round" icon-md="f7:play_round" icon-aurora="f7:play_round" icon-size="32" :color="(rule.status.status === 'IDLE') ? 'blue' : 'gray'" @click="runNow" />
            </div>
            Status:
            <f7-chip class="margin-left"
                     :text="rule.status.status"
                     :color="ruleStatusBadgeColor(rule.status)" />
            <div>
              <strong>{{ (rule.status.statusDetail !== 'NONE') ? rule.status.statusDetail : '&nbsp;' }}</strong>
              <br>
              <div v-if="rule.status.description">
                {{ rule.status.description }}
              </div>
            </div>
          </f7-col>
        </f7-block>
        <!-- skeletons for not ready -->
        <f7-block v-else-if="!createMode && !stubMode" class="block-narrow padding-left padding-right skeleton-text skeleton-effect-blink" strong>
          <f7-col>
            ______:
            <f7-chip class="margin-left" text="________" />
            <div>
              <strong>____ _______</strong>
              <br>
            </div>
          </f7-col>
        </f7-block>

        <rule-general-settings :rule="rule" :ready="ready" :createMode="createMode" :stubMode="stubMode" :templateName="templateName" />

        <f7-block v-if="ready" class="block-narrow">
          <f7-block-footer v-if="!isEditable" class="no-margin padding-left">
            <f7-icon f7="lock_fill" size="12" color="gray" />&nbsp;Note: this rule is not editable.
          </f7-block-footer>
          <!-- <f7-col v-if="isEditable" class="text-align-right justify-content-flex-end">
          </f7-col> -->
          <f7-col v-if="createMode && templates.length > 0 && !ruleCopy" class="new-rule-from-template">
            <f7-block-title medium class="margin-bottom">
              Create from Template
            </f7-block-title>
            <f7-list media-list>
              <f7-list-item title="No template" footer="Create a new rule from scratch"
                            radio :checked="!hasTemplate" radio-icon="start"
                            :value="''"
                            @change="selectTemplate(null)" />
            </f7-list>
            <f7-block-footer class="margin-left">
              or choose a rule template:
            </f7-block-footer>
            <f7-list media-list>
              <f7-list-item :key="template.uid" v-for="template in templates"
                            :title="template.label" :footer="template.description"
                            :value="template.uid"
                            radio :checked="hasTemplate && currentTemplate.uid === template.uid" radio-icon="start"
                            @change="selectTemplate(template.uid)" />
            </f7-list>
            <f7-block-title v-if="hasTemplate" medium class="margin-vertical padding-top">
              Template Configuration
            </f7-block-title>
            <f7-link v-if="templateTopicLink" target="_blank" class="external margin-left" color="blue" :href="templateTopicLink">
              Template Community Marketplace Topic
            </f7-link>
            <config-sheet v-if="hasTemplate"
                          :parameter-groups="[]" :parameters="currentTemplate.configDescriptions"
                          :configuration="rule.configuration" />
          </f7-col>
          <f7-col v-else-if="currentTemplate && stubMode" class="show-associated-template">
            <f7-block-title medium class="margin-vertical padding-top">
              Template
            </f7-block-title>
            <f7-list media-list>
              <f7-list-item :title="currentTemplate.label" :footer="currentTemplate.description" :value="currentTemplate.uid" />
            </f7-list>
            <f7-block-title medium class="margin-vertical padding-top">
              Template Configuration
            </f7-block-title>
            <f7-link v-if="templateTopicLink" target="_blank" class="external margin-left" color="blue" :href="templateTopicLink">
              Template Community Marketplace Topic
            </f7-link>
            <config-sheet :parameter-groups="[]" :parameters="currentTemplate.configDescriptions" :configuration="rule.configuration" />
          </f7-col>
          <f7-col v-else-if="currentTemplate && createMode && ruleCopy?.templateUID" class="select-integrate-template">
            <f7-block-title medium class="margin-vertical padding-top">
              Template
            </f7-block-title>
            <f7-list media-list>
              <f7-list-item
                :title="'Keep template: ' + currentTemplate.label"
                footer="The rule will still be linked to the template and can be regenerated if the template changes."
                :value="currentTemplate.uid"
                radio :checked="Boolean(rule.templateUID)" radio-icon="start"
                @change="keepTemplate(true)" />
              <f7-list-item
                title="Integrate template"
                footer="Integrates the template in the rule so that the rule is no longer linked to the template."
                value="integrate"
                radio :checked="!rule.templateUID" radio-icon="start"
                @change="keepTemplate(false)" />
            </f7-list>
            <div v-if="rule.templateUID">
              <f7-block-title medium class="margin-vertical padding-top">
                Template Configuration
              </f7-block-title>
              <f7-link v-if="templateTopicLink" target="_blank" class="external margin-left" color="blue" :href="templateTopicLink">
                Template Community Marketplace Topic
              </f7-link>
              <config-sheet :parameter-groups="[]" :parameters="currentTemplate.configDescriptions" :configuration="rule.configuration" />
            </div>
          </f7-col>
          <f7-col v-if="!hasTemplate || (createMode && ruleCopy?.templateUID && !rule.templateUID)" class="rule-modules">
            <div v-if="isEditable" class="no-padding float-right">
              <f7-button @click="toggleModuleControls" small outline :fill="showModuleControls" sortable-toggle=".sortable" style="margin-top: -3px; margin-right: 5px"
                         color="gray" icon-size="12" icon-ios="material:wrap_text" icon-md="material:wrap_text" icon-aurora="material:wrap_text">
                &nbsp;Reorder
              </f7-button>
            </div>
            <div v-for="section in ['triggers', 'actions', 'conditions']" :key="section">
              <template v-if="isEditable || rule[section].length > 0">
                <f7-block-title medium class="no-margin-bottom">
                  {{ SECTION_LABELS[section][0] }}
                </f7-block-title>
                <f7-block-footer class="no-margin-top margin-horizontal" style="margin-bottom: var(--f7-list-margin-vertical)">
                  {{ SECTION_LABELS[section][1] }}
                </f7-block-footer>
              </template>
              <f7-list sortable swipeout media-list @sortable:sort="(ev) => reorderModule(ev, section)">
                <f7-list-item media
                              :title="mod.label || suggestedModuleTitle(mod, null, section)"
                              :footer="mod.description || suggestedModuleDescription(mod, null, section)"
                              v-for="mod in rule[section]" :key="mod.id"
                              :link="!showModuleControls && !isOpaqueModule(mod)"
                              @click.native="(ev) => editModule(ev, section, mod)" swipeout>
                  <f7-link slot="media" v-if="isEditable" icon-color="red" icon-aurora="f7:minus_circle_filled" icon-ios="f7:minus_circle_filled" icon-md="material:remove_circle_outline" @click="showSwipeout" />
                  <f7-swipeout-actions right v-if="isEditable">
                    <f7-swipeout-button @click="(ev) => deleteModule(ev, section, mod)" style="background-color: var(--f7-swipeout-delete-button-bg-color)">
                      Delete
                    </f7-swipeout-button>
                  </f7-swipeout-actions>
                </f7-list-item>
              </f7-list>
              <f7-list v-if="isEditable">
                <f7-list-item link no-chevron media-item :color="($theme.dark) ? 'black' : 'white'" :subtitle="SECTION_LABELS[section][2]" @click="addModule(section)">
                  <f7-icon slot="media" color="green" aurora="f7:plus_circle_fill" ios="f7:plus_circle_fill" md="material:control_point" />
                </f7-list-item>
                <!-- <f7-list-button :color="(showModuleControls) ? 'gray' : 'blue'" :title="sectionLabels[section][1]"></f7-list-button> -->
              </f7-list>
            </div>
          </f7-col>
          <f7-col v-if="!createMode && !stubMode">
            <f7-list>
              <f7-list-button v-if="isEditable || !hasOpaqueModule" color="blue" @click="duplicateRule">
                Duplicate Rule
              </f7-list-button>
              <f7-list-button v-if="isEditable" color="red" @click="deleteRule">
                Delete Rule
              </f7-list-button>
            </f7-list>
          </f7-col>
        </f7-block>
      </f7-tab>
      <f7-tab id="code" @tab:show="() => { this.currentTab = 'code'; toYaml() }" :tab-active="currentTab === 'code'">
        <f7-icon v-if="!createMode && !isEditable" f7="lock" class="float-right margin" style="opacity:0.5; z-index: 4000; user-select: none;" size="50" color="gray" tooltip="This code is not editable" />
        <editor v-if="currentTab === 'code'" class="rule-code-editor" mode="application/vnd.openhab.rule+yaml" :value="ruleYaml" :readOnly="!isEditable" @input="onEditorInput" />
        <!-- <pre class="yaml-message padding-horizontal" :class="[yamlError === 'OK' ? 'text-color-green' : 'text-color-red']">{{yamlError}}</pre> -->
      </f7-tab>
      <f7-tab v-if="ready && hasSource" id="source" @tab:show="() => { this.currentTab = 'source'} " :tab-active="currentTab === 'source'">
        <f7-icon f7="lock" class="float-right margin" style="opacity:0.5; z-index: 4000; user-select: none;" size="50" color="gray" tooltip="Source code is not editable" />
        <editor v-if="currentTab === 'source'" class="rule-source-viewer" :mode="sourceType" :value="source" :readOnly="true" />
      </f7-tab>
    </f7-tabs>
  </f7-page>
</template>

<style lang="stylus">
.enable-toggle
  vertical-align inherit
.moduleconfig-popup
  .page-content
    overflow-x hidden !important
  .config-sheet, .parameter-group
    margin-top 0 !important
.rule-modules
  .swipeout-opened
    .sortable-handler
      display none
  .item-media .icon
    color var(--f7-theme-color)
  .media-list
    margin-bottom 0
  .list
    margin-top 0
.rule-code-editor.vue-codemirror
  display block
  top calc(var(--f7-navbar-height) + var(--f7-tabbar-height))
  height calc(100% - 2*var(--f7-navbar-height))
  width 100%
.yaml-message
  display block
  position absolute
  top 80%
  white-space pre-wrap
#source
  .rule-source-viewer.vue-codemirror
    display block
    top calc(var(--f7-navbar-height) + var(--f7-tabbar-height))
    height calc(100% - 2*var(--f7-navbar-height))
    width 100%
</style>

<script>
import YAML from 'yaml'
import cloneDeep from 'lodash/cloneDeep'
import fastDeepEqual from 'fast-deep-equal/es6'

import RuleModulePopup from './rule-module-popup.vue'

import RuleMixin from './rule-edit-mixin'
import ModuleDescriptionSuggestions from './module-description-suggestions'
import RuleStatus from '@/components/rule/rule-status-mixin'
import DirtyMixin from '../dirty-mixin'

import ConfigSheet from '@/components/config/config-sheet.vue'
import RuleGeneralSettings from '@/components/rule/rule-general-settings.vue'
import AUTOMATION_LANGUAGES from '@/assets/automation-languages'

export default {
  mixins: [RuleMixin, ModuleDescriptionSuggestions, RuleStatus, DirtyMixin],
  components: {
    RuleGeneralSettings,
    ConfigSheet,
    'editor': () => import(/* webpackChunkName: "script-editor" */ '@/components/config/controls/script-editor.vue')
  },
  props: ['ruleId', 'createMode', 'ruleCopy', 'stubMode', 'schedule'],
  data () {
    return {
      SECTION_LABELS: {
        triggers: ['When', 'Events that cause this rule to run', 'Add Trigger'],
        actions: ['Then', 'Actions to take when this rule runs', 'Add Action'],
        conditions: ['But only if', 'Conditions that must be matched for the actions of this rule to run', 'Add Condition']
      },

      ready: false,
      loading: false,

      rule: {},
      savedRule: {},
      ruleYaml: '',
      moduleTypes: {
        actions: [],
        conditions: [],
        triggers: []
      },
      currentSection: 'actions',
      currentModuleType: null,
      currentModule: null,
      currentModuleConfig: {},

      currentTab: 'design',
      codeEditorOpened: false,
      cronPopupOpened: false,
      scriptCode: '',
      cronExpression: null,
      templates: null,
      currentTemplate: null
    }
  },
  watch: {
    rule: {
      handler: function () {
        if (!this.loading) { // ignore changes during loading
          // create rule object clone in order to be able to delete status part
          // which can change from eventsource but doesn't mean a rule modification
          let ruleClone = cloneDeep(this.rule)
          delete ruleClone.status
          delete this.savedRule.status

          this.dirty = !fastDeepEqual(ruleClone, this.savedRule)
        }
      },
      deep: true
    }
  },
  methods: {
    load () {
      if (this.loading) return
      this.loading = true

      const loadingFinished = () => {
        this.$nextTick(() => {
          this.savedRule = cloneDeep(this.rule)
          this.ready = true
          this.loading = false
          if (!this.createMode && !this.stubMode && this.hasOpaqueModule && this.hasSource) {
            this.switchTab('source')
          }
        })
      }

      this.$oh.api.get('/rest/module-types?asMap=true').then((data) => {
        this.moduleTypes = data
        if (this.createMode) {
          let newRule
          if (this.ruleCopy) {
            newRule = cloneDeep(this.ruleCopy)
            newRule.uid = this.$f7.utils.id()
            if (newRule.templateUID) {
              newRule.triggers = []
              newRule.actions = []
              newRule.conditions = []
              if (newRule.templateState === 'instantiated') {
                newRule.templateState = 'pending'
              }
            }
          } else {
            newRule = {
              uid: this.$f7.utils.id(),
              name: '',
              triggers: [],
              actions: [],
              conditions: [],
              tags: (this.schedule) ? ['Schedule'] : [],
              configuration: {},
              templateUID: null,
              visibility: 'VISIBLE',
              status: {
                status: 'NEW'
              }
            }
          }
          this.$set(this, 'rule', newRule)
          this.$oh.api.get('/rest/templates').then((templateData) => {
            this.$set(this, 'templates', templateData)
            if (newRule.templateUID) {
              const currentTemplate = templateData.find((t) => t.uid === newRule.templateUID) || {
                uid: newRule.templateUID,
                label: newRule.templateUID
              }
              this.$set(this, 'currentTemplate', currentTemplate)
            }
            loadingFinished()
          })
          // no need for an event source, the rule doesn't exist yet
        } else if (this.stubMode) {
          if (!this.ruleCopy || !this.ruleCopy.templateUID) {
            this.$f7.toast.create({
              text: !this.ruleCopy ? 'Failed to create rule stub because there\'s no source rule' : 'Failed to create rule stub because there\'s no template UID',
              destroyOnClose: true,
              closeTimeout: 4000
            }).open()
            this.$f7router.back()
          }
          const ruleStub = this.ruleCopy
          ruleStub.triggers = []
          ruleStub.actions = []
          ruleStub.conditions = []
          ruleStub.templateState = 'pending'
          this.$set(this, 'rule', ruleStub)
          this.$oh.api.get('/rest/templates').then((templateData) => {
            this.$set(this, 'templates', templateData)
            let template = this.templates.find((t) => t.uid === ruleStub.templateUID)
            if (!template) {
              this.$f7.toast.create({
                text: 'Template "' + ruleStub.templateUID + '" not found',
                destroyOnClose: true,
                closeTimeout: 4000
              }).open()
              this.$f7router.back()
            }
            this.$set(this, 'currentTemplate', template)
            loadingFinished()
          })
          // no need for an event source, we're going to overwrite the existing rule
        } else {
          this.$oh.api.get('/rest/rules/' + this.ruleId).then((data2) => {
            this.$set(this, 'rule', data2)
            if (data2.templateUID) {
              this.$oh.api.get('/rest/templates').then((templateData) => {
                this.$set(this, 'templates', templateData)
                if (!this.eventSource) this.startEventSource()
                loadingFinished()
              })
            } else {
              if (!this.eventSource) this.startEventSource()
              loadingFinished()
            }
          })
        }
      })
    },
    save (noToast) {
      if (!this.isEditable) return Promise.reject()
      if (this.currentTab === 'code') {
        if (!this.fromYaml()) {
          return Promise.reject()
        }
      }
      if (!this.rule.uid) {
        this.$f7.dialog.alert('Please give an ID to the rule')
        return Promise.reject()
      }
      if (!this.rule.name) {
        this.$f7.dialog.alert('Please give a name to the rule')
        return Promise.reject()
      }
      const promise = (this.createMode)
        ? this.$oh.api.postPlain('/rest/rules', JSON.stringify(this.rule), 'text/plain', 'application/json')
        : this.$oh.api.put('/rest/rules/' + this.rule.uid, this.rule)
      return promise.then((data) => {
        this.dirty = false
        if (this.createMode) {
          this.$f7.toast.create({
            text: 'Rule created',
            destroyOnClose: true,
            closeTimeout: 2000
          }).open()
          this.$f7router.navigate(this.$f7route.url
            .replace('/add', '/' + this.rule.uid)
            .replace('/duplicate', '/' + this.rule.uid)
            .replace('/schedule/', '/rules/'), { reloadCurrent: true })
          this.load()
        } else if (this.stubMode) {
          this.$f7.toast.create({
            text: 'Rule regenerated',
            destroyOnClose: true,
            closeTimeout: 2000
          }).open()
          this.$f7router.navigate(this.$f7route.url
            .replace('/stub', '/' + this.rule.uid)
            .replace('/schedule/', '/rules/'), { reloadCurrent: true })
          this.load()
        } else {
          if (!noToast) {
            this.$f7.toast.create({
              text: 'Rule updated',
              destroyOnClose: true,
              closeTimeout: 2000
            }).open()
          }
          this.savedRule = cloneDeep(this.rule)
        }
        // if (!stay) this.$f7router.back()
      }).catch((err) => {
        this.$f7.toast.create({
          text: 'Error while saving rule: ' + err,
          destroyOnClose: true,
          closeTimeout: 2000
        }).open()
      })
    },
    duplicateRule () {
      let ruleClone = cloneDeep(this.rule)
      ruleClone.name = (ruleClone.name || '') + ' copy'
      ruleClone.editable = true
      this.$f7router.navigate({
        url: '/settings/rules/duplicate'
      }, {
        props: {
          ruleCopy: ruleClone
        }
      })
    },
    regenerateFromTemplate () {
      if (this.isEditable) {
        this.createStub()
      } else {
        this.$oh.api.postPlain('/rest/rules/' + this.rule.uid + '/regenerate').then(() => {
          this.$f7.toast.create({
            text: 'Rule regenerated from template',
            destroyOnClose: true,
            closeTimeout: 2000
          }).open()
          this.load()
        }).catch((err) => {
          this.$f7.dialog.alert('An error occurred when trying to regenerate rule "' + this.rule.uid + '" from template: ' + err)
        })
      }
    },
    createStub () {
      let ruleClone = cloneDeep(this.rule)
      this.$f7router.navigate({
        url: '/settings/rules/stub'
      }, {
        reloadCurrent: true,
        props: {
          ruleCopy: ruleClone
        }
      })
    },
    runNow () {
      if (this.createMode) return
      if (this.rule.status.status === 'RUNNING' || this.rule.status.status === 'UNINITIALIZED') {
        return this.$f7.toast.create({
          text: `Rule cannot be run ${(this.rule.status.status === 'RUNNING') ? 'while already running, please wait' : 'if it is uninitialized'}!`,
          destroyOnClose: true,
          closeTimeout: 2000
        }).open()
      }
      this.$f7.toast.create({
        text: 'Running rule',
        destroyOnClose: true,
        closeTimeout: 2000
      }).open()

      const savePromise = (this.isEditable && this.dirty) ? this.save(true) : Promise.resolve()

      savePromise.then(() => {
        this.$oh.api.postPlain('/rest/rules/' + this.rule.uid + '/runnow', '').catch((err) => {
          this.$f7.toast.create({
            text: 'Error while running rule: ' + err,
            destroyOnClose: true,
            closeTimeout: 2000
          }).open()
        })
      })
    },
    deleteRule () {
      this.$f7.dialog.confirm(
        `Are you sure you want to delete ${this.rule.name}?`,
        'Delete Rule',
        () => {
          this.$oh.api.delete('/rest/rules/' + this.rule.uid).then(() => {
            this.dirty = false
            this.$f7router.back('/settings/rules/', { force: true })
          })
        }
      )
    },
    selectTemplate (uid) {
      this.$set(this.rule, 'configuration', {})
      this.$set(this.rule, 'triggers', [])
      this.$set(this.rule, 'conditions', [])
      this.$set(this.rule, 'actions', [])
      if (!uid) {
        this.$set(this, 'currentTemplate', null)
        return
      }
      this.$set(this, 'currentTemplate', this.templates.find((t) => t.uid === uid))
      this.rule.templateUID = uid
      this.rule.templateState = 'pending'
    },
    keepTemplate (keep) {
      if (!this.ruleCopy) return
      let newRule = this.rule
      if (keep) {
        newRule.triggers = []
        newRule.actions = []
        newRule.conditions = []
        newRule.configuration = this.ruleCopy.configuration
        newRule.templateUID = this.ruleCopy.templateUID
        newRule.templateState = 'pending'
        if (!newRule.tags?.some((t) => t.indexOf('marketplace:') === 0)) {
          const tag = this.ruleCopy.tags?.find((t) => t.indexOf('marketplace:') === 0)
          if (tag) {
            if (!newRule.tags) {
              newRule.tags = [tag]
            } else {
              newRule.tags.push(tag)
            }
          }
        }
      } else {
        newRule.triggers = this.ruleCopy.triggers
        newRule.actions = this.ruleCopy.actions
        newRule.conditions = this.ruleCopy.conditions
        newRule.configuration = {}
        newRule.templateUID = null
        newRule.templateState = 'no-template'
        if (newRule.tags) {
          newRule.tags = newRule.tags.filter((t) => t.indexOf('marketplace:') !== 0)
        }
      }
      this.$set(this, 'rule', newRule)
    },
    editModule (ev, section, mod) {
      if (this.showModuleControls || this.isOpaqueModule(mod)) return
      let swipeoutElement = ev.target
      ev.cancelBubble = true
      while (!swipeoutElement.classList.contains('swipeout')) {
        swipeoutElement = swipeoutElement.parentElement
      }
      if (swipeoutElement && swipeoutElement.classList.contains('swipeout-opened')) return

      if (mod.type && mod.type.indexOf('script') === 0) {
        this.editScriptDirect(ev, mod)
        return
      }

      this.currentSection = section
      this.currentModule = Object.assign({}, mod)
      if (!this.currentModule.label) this.currentModule.label = ''
      if (!this.currentModule.description) this.currentModule.description = ''
      this.currentModuleType = this.moduleTypes[section].find((m) => m.uid === mod.type)

      const popup = {
        component: RuleModulePopup
      }
      this.$f7router.navigate({
        url: 'module-config',
        route: {
          path: 'module-config',
          popup
        }
      }, {
        props: {
          rule: this.rule,
          currentSection: this.currentSection,
          ruleModule: this.currentModule,
          ruleModuleType: this.currentModuleType,
          moduleTypes: this.moduleTypes,
          readOnly: !this.isEditable
        }
      })

      if (this.isEditable) {
        this.$f7.once('ruleModuleConfigUpdate', this.saveModule)
        this.$f7.once('ruleModuleConfigClosed', () => {
          this.$f7.off('ruleModuleConfigUpdate', this.saveModule)
          this.moduleConfigClosed()
        })
      }
    },
    deleteModule (ev, section, mod) {
      let swipeoutElement = ev.target
      if (!this.isEditable) return
      ev.cancelBubble = true
      while (!swipeoutElement.classList.contains('swipeout')) {
        swipeoutElement = swipeoutElement.parentElement
      }
      this.$f7.swipeout.delete(swipeoutElement, () => {
        const idx = this.rule[section].findIndex((m) => m.id === mod.id)
        this.rule[section].splice(idx, 1)
      })
    },
    addModule (section) {
      if (this.showModuleControls) return
      if (!this.isEditable) return
      let moduleId = 1
      for (; ['triggers', 'actions', 'conditions'].some((s) => this.rule[s].some((m) => m.id === moduleId.toString())); moduleId++);
      console.debug('new moduleId=' + moduleId)
      const newModule = {
        id: moduleId.toString(),
        configuration: {},
        description: '',
        label: '',
        type: '',
        new: true
      }

      // this.rule[section].push(newModule)
      this.currentSection = section
      this.currentModule = newModule
      this.currentModuleType = null

      const popup = {
        component: RuleModulePopup
      }
      this.$f7router.navigate({
        url: 'module-config',
        route: {
          path: 'module-config',
          popup
        }
      }, {
        props: {
          currentSection: this.currentSection,
          ruleModule: this.currentModule,
          ruleModuleType: this.currentModuleType,
          moduleTypes: this.moduleTypes
        }
      })

      this.$f7.once('ruleModuleConfigUpdate', this.saveModule)
      this.$f7.once('editNewScript', this.saveAndEditNewScript)
      this.$f7.once('ruleModuleConfigClosed', () => {
        this.$f7.off('ruleModuleConfigUpdate', this.saveModule)
        this.$f7.off('editNewScript', this.saveAndEditNewScript)
        this.moduleConfigClosed()
      })
    },
    reorderModule (ev, section) {
      const newSection = [...this.rule[section]]
      newSection.splice(ev.to, 0, newSection.splice(ev.from, 1)[0])
      this.$set(this.rule, section, newSection)
    },
    saveModule (updatedModule) {
      if (!updatedModule.type) return
      if (!updatedModule.label) delete updatedModule.label
      if (!updatedModule.description) delete updatedModule.description
      if (updatedModule.new) {
        delete updatedModule.new
        this.rule[this.currentSection].push(updatedModule)
      } else {
        const idx = this.rule[this.currentSection].findIndex((m) => m.id === updatedModule.id)
        this.$set(this.rule[this.currentSection], idx, updatedModule)
      }
    },
    saveAndEditNewScript (updatedModule) {
      this.saveModule(updatedModule)
      this.save().then(() => {
        this.$f7router.navigate('/settings/rules/' + this.rule.uid + '/script/' + updatedModule.id, { transition: this.$theme.aurora ? 'f7-cover-v' : '' })
      })
    },
    moduleConfigClosed () {
      this.currentModule = null
      this.currentModuleType = null
    },
    editScriptDirect (ev, mod) {
      ev.cancelBubble = true
      this.currentModule = mod
      this.currentModuleType = mod.type
      this.scriptCode = mod.configuration.script

      const updatePromise = (this.rule.editable || this.createMode) && this.dirty ? this.save() : Promise.resolve()
      updatePromise.then(() => {
        this.$f7router.navigate('/settings/rules/' + this.rule.uid + '/script/' + mod.id, { transition: this.$theme.aurora ? 'f7-cover-v' : '' })
      })
    },
    toYaml () {
      this.ruleYaml = YAML.stringify({
        configuration: this.rule.configuration,
        triggers: this.rule.triggers,
        conditions: this.rule.conditions,
        actions: this.rule.actions
      }, this.isEditable ? undefined : this.replacer)
    },
    fromYaml () {
      if (!this.isEditable || !this.ruleYaml) return
      try {
        const updatedRule = YAML.parse(this.ruleYaml)
        this.$set(this.rule, 'configuration', updatedRule.configuration)
        this.$set(this.rule, 'triggers', updatedRule.triggers)
        this.$set(this.rule, 'conditions', updatedRule.conditions)
        this.$set(this.rule, 'actions', updatedRule.actions)
        return true
      } catch (e) {
        this.$f7.dialog.alert(e).open()
        return false
      }
    },
    /**
     * Replaces CRLF (Windows) or CR (Mac) with LF in scripts before YAMLification.
     *
     * @param key the key being processed
     * @param value the value being processed
     */
    replacer (key, value) {
      switch (key) {
        case 'script':
          return value ? value.replaceAll(/(\r\n|\r)/g, '\n') : value
        default:
          return value
      }
    },
    /**
     * Determines if the module is "opaque" in that it doesn't actually execute the content of the module, but instead executes
     * a referenced in-memory runnable method.
     *
     * @param module the module to evaluate
     */
    isOpaqueModule (module) {
      if (!module?.type) return false
      return module.type === 'jsr223.ScriptedAction' || module.type === 'jsr223.ScriptedCondition' || module.type === 'jsr223.ScriptedTrigger'
    }
  },
  computed: {
    hasTemplate () {
      return this.rule && (this.stubMode || this.currentTemplate !== null)
    },
    templateName () {
      if (!this.rule || !this.rule.templateUID || !this.templates) {
        return undefined
      }
      let result = this.templates.find((t) => t.uid === this.rule.templateUID)
      return result ? result.label : this.rule.templateUID
    },
    canRegenerate () {
      if (!this.rule || !this.rule.templateUID || !this.rule.templateState || this.rule.templateState === 'no-template' || this.rule.templateState === 'template-missing') {
        return false
      }
      return this.templates ? this.templates.some((t) => t.uid === this.rule.templateUID) : false
    },
    hasOpaqueModule () {
      return this.opaqueModules.length > 0
    },
    opaqueModulesTypeText () {
      const result = this.opaqueModulesType
      return result ? AUTOMATION_LANGUAGES[result]?.name || result : result
    },
    opaqueModulesType () {
      const modules = this.opaqueModules
      if (!modules || !modules.length) return undefined
      // "Opaque modules" implies that the rule is created through JSR223.
      // The assumption is therefore that all opaque module types are of the same type/scripting language.
      return modules.find((m) => m.configuration?.type)?.configuration?.type
    },
    opaqueModules () {
      if (!this.rule) return []
      return [...this.rule.actions || [], this.rule.triggers || [], this.rule.conditions || []].filter((m) => this.isOpaqueModule(m))
    },
    hasSource () {
      const sourceContainer = this.sourceSource
      return sourceContainer ? sourceContainer.source || sourceContainer.script : false
    },
    source () {
      const sourceContainer = this.sourceSource
      if (!sourceContainer) return ''
      return sourceContainer.source || sourceContainer.script || ''
    },
    sourceTypeText () {
      const result = this.sourceType
      return result ? AUTOMATION_LANGUAGES[result]?.name || result : result
    },
    sourceType () {
      const sourceContainer = this.sourceSource
      return sourceContainer ? sourceContainer.sourceType || sourceContainer.type : undefined
    },
    sourceSource () {
      if (!this.rule) return undefined
      if (this.rule.configuration?.source) {
        return this.rule.configuration
      }
      if (this.rule.actions?.length) {
        for (const action of this.rule.actions) {
          if (this.isOpaqueModule(action)) {
            return action.configuration
          }
        }
      }
      return undefined
    },
    templateTopicLink () {
      if (!this.currentTemplate) return null
      if (!this.currentTemplate.tags) return null
      const marketplaceTag = this.currentTemplate.tags.find((t) => t.indexOf('marketplace:') === 0)
      if (marketplaceTag) return 'https://community.openhab.org/t/' + marketplaceTag.replace('marketplace:', '')
      return null
    }
  }
}
</script>
