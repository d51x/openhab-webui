<template>
  <f7-page @page:afterin="onPageAfterIn" @page:beforeout="onPageBeforeOut" class="home-editor">
    <f7-navbar title="Edit Home Page" back-link="Back" no-hairline>
      <f7-nav-right>
        <f7-link @click="save()" v-if="$theme.md" icon-md="material:save" icon-only />
        <f7-link @click="save()" v-if="!$theme.md">
          Save<span v-if="$device.desktop">&nbsp;(Ctrl-S)</span>
        </f7-link>
      </f7-nav-right>
    </f7-navbar>
    <f7-toolbar tabbar position="top" v-if="!previewMode">
      <f7-link @click="switchTab('design', fromYaml)" :tab-link-active="currentTab === 'design'" class="tab-link">
        Design
      </f7-link>
      <f7-link @click="switchTab('code', toYaml)" :tab-link-active="currentTab === 'code'" class="tab-link">
        Code
      </f7-link>
    </f7-toolbar>
    <f7-toolbar v-else tabbar position="top">
      <f7-link v-for="tab in modelTabs" :key="tab.value" @click="showCardControls = false; currentModelTab = tab.value" :tab-link-active="currentModelTab === tab.value" class="tab-link">
        {{ tab.label }}
      </f7-link>
    </f7-toolbar>
    <f7-toolbar bottom class="toolbar-details">
      <div style="margin-left: auto">
        <f7-toggle :checked="previewMode" @toggle:change="(value) => togglePreviewMode(value)" /> Run mode<span v-if="$device.desktop">&nbsp;(Ctrl-R)</span>
      </div>
    </f7-toolbar>
    <f7-tabs class="tabs-editor-tabs">
      <f7-tab id="design" class="tabs-editor-design-tab" @tab:show="() => this.currentTab = 'design'" :tab-active="currentTab === 'design'">
        <f7-block v-if="!ready || !modelReady" class="text-align-center">
          <f7-preloader />
          <div>Loading...</div>
        </f7-block>
        <!-- <f7-block class="block-narrow" v-if="ready && !previewMode">
          <page-settings :page="page" :createMode="createMode" />
        </f7-block> -->

        <f7-block class="block-narrow" style="padding-bottom: 8rem" v-else-if="ready && modelReady && !previewMode">
          <f7-col>
            <f7-block-title>Page Configuration</f7-block-title>
            <config-sheet
              :parameterGroups="pageWidgetDefinition.props.parameterGroups || []"
              :parameters="pageWidgetDefinition.props.parameters || []"
              :configuration="page.config"
              @updated="dirty = true" />
          </f7-col>

          <f7-col>
            <f7-segmented strong tag="p">
              <f7-button v-for="tab in modelTabs" :key="tab.value" @click="showCardControls = false; currentModelTab = tab.value" :active="currentModelTab === tab.value" :text="tab.label" />
            </f7-segmented>

            <f7-block-title>Cards</f7-block-title>
            <div>
              <div class="display-block padding">
                <div class="no-padding float-right">
                  <f7-button @click="showCardControls = !showCardControls" small outline :fill="showCardControls" sortable-toggle=".sortable" style="margin-top: -3px; margin-right: 5px"
                             color="gray" icon-size="12" icon-ios="material:wrap_text" icon-md="material:wrap_text" icon-aurora="material:wrap_text">
                    &nbsp;Reorder
                  </f7-button>
                </div>
              </div>

              <f7-list media-list class="homecards-list" sortable :key="'cards-' + currentModelTab + cardListId" @sortable:sort="reorderCard">
                <f7-list-item media-item :link="(showCardControls) ? undefined : ''"
                              @click.native="(ev) => cardClicked(ev, card, idx)"
                              v-for="(card, idx) in cardGroups(currentModelTab, page).flat()" :key="idx"
                              :title="card.separator || card.defaultTitle" :footer="(card.separator) ? '(separator)' : card.key">
                  <f7-menu slot="content-start" class="configure-layout-menu">
                    <f7-menu-item icon-f7="list_bullet" dropdown>
                      <f7-menu-dropdown>
                        <f7-menu-dropdown-item v-if="!card.separator" @click="configureCard(card)" href="#" text="Configure Card" />
                        <f7-menu-dropdown-item v-if="!card.separator" @click="editCardCode(card)" href="#" text="Edit YAML" />
                        <f7-menu-dropdown-item v-if="card.separator" @click="renameCardSeparator(idx)" href="#" text="Rename" />
                        <f7-menu-dropdown-item divider />
                        <f7-menu-dropdown-item v-if="!card.separator" @click="addCardSeparator(idx)" href="#" text="Add Separator Before" />
                        <f7-menu-dropdown-item v-if="card.separator" @click="removeCardSeparator(idx)" href="#" text="Remove Separator" />
                      </f7-menu-dropdown>
                    </f7-menu-item>
                  </f7-menu>
                  <f7-checkbox :checked="!isCardExcluded(card)" :disabled="card.separator !== undefined" slot="content-start" class="margin-right" />
                </f7-list-item>
              </f7-list>
            </div>

            <div v-if="currentModelTab === 'locations'">
              <config-sheet
                :parameterGroups="locationsTabParameters.props.parameterGroups || []"
                :parameters="locationsTabParameters.props.parameters || []"
                :configuration="page.slots.locations[0].config"
                @updated="dirty = true" />
            </div>

            <div v-if="currentModelTab === 'equipment'">
              <config-sheet
                :parameterGroups="equipmentTabParameters.props.parameterGroups || []"
                :parameters="equipmentTabParameters.props.parameters || []"
                :configuration="page.slots.equipment[0].config"
                @updated="dirty = true" />
            </div>

            <div v-if="currentModelTab === 'properties'">
              <config-sheet
                :parameterGroups="propertiesTabParameters.props.parameterGroups || []"
                :parameters="propertiesTabParameters.props.parameters || []"
                :configuration="page.slots.properties[0].config"
                @updated="dirty = true" />
            </div>
          </f7-col>
        </f7-block>
        <div v-else-if="ready && previewMode && currentTab === 'design'" :context="context" :key="pageKey">
          <model-tab style="margin-bottom: 4rem" :context="context" :type="currentModelTab" :model="model" :page="page" />
        </div>
      </f7-tab>

      <f7-tab id="code" @tab:show="() => { this.currentTab = 'code' }" :tab-active="currentTab === 'code'">
        <editor v-if="currentTab === 'code'" :style="{ opacity: previewMode ? '0' : '' }" class="page-code-editor" mode="application/vnd.openhab.uicomponent+yaml;type=home" :value="pageYaml" @input="onEditorInput" />
        <!-- <pre class="yaml-message padding-horizontal" :class="[yamlError === 'OK' ? 'text-color-green' : 'text-color-red']">{{yamlError}}</pre> -->
        <div v-if="ready && previewMode" :context="context" :key="pageKey">
          <model-tab style="margin-bottom: 4rem" :context="context" :type="currentModelTab" :model="model" :page="page" />
        </div>
      </f7-tab>
    </f7-tabs>
  </f7-page>
</template>

<style lang="stylus">
.page-code-editor.vue-codemirror
  display block
  top calc(var(--f7-navbar-height) + var(--f7-tabbar-height))
  height calc(100% - 3*var(--f7-navbar-height))
  width 100%
.yaml-message
  display block
  position absolute
  top 80%
  white-space pre-wrap
.homecards-list
  .item-link
    overflow inherit
    z-index inherit !important
</style>

<script>
import PageDesigner from '../pagedesigner-mixin'
import HomeCards from '../../../home/homecards-mixin'

import YAML from 'yaml'

import { OhHomePageDefinition, OhLocationsTabParameters, OhEquipmentTabParameters, OhPropertiesTabParameters, OhLocationCardParameters, OhEquipmentCardParameters, OhPropertyCardParameters } from '@/assets/definitions/widgets/home'

import ConfigSheet from '@/components/config/config-sheet.vue'
import ModelTab from '@/pages/home/model-tab.vue'

const ConfigurableWidgets = {
  'oh-location-card': OhLocationCardParameters,
  'oh-equipment-card': OhEquipmentCardParameters,
  'oh-property-card': OhPropertyCardParameters
}

export default {
  mixins: [PageDesigner, HomeCards],
  components: {
    'editor': () => import(/* webpackChunkName: "script-editor" */ '@/components/config/controls/script-editor.vue'),
    ConfigSheet,
    ModelTab
  },
  props: ['createMode', 'uid'],
  data () {
    return {
      pageWidgetDefinition: OhHomePageDefinition(),
      locationsTabParameters: OhLocationsTabParameters(),
      equipmentTabParameters: OhEquipmentTabParameters(),
      propertiesTabParameters: OhPropertiesTabParameters(),
      currentModelTab: 'locations',
      modelTabs: [],
      showCardControls: false,
      cardListId: this.$f7.utils.id(),
      page: {
        uid: 'home',
        component: 'oh-home-page',
        config: {
          label: 'Home Page'
        },
        slots: {
          locations: [{ component: 'oh-locations-tab', config: {}, slots: {} }],
          equipment: [{ component: 'oh-equipment-tab', config: {}, slots: {} }],
          properties: [{ component: 'oh-properties-tab', config: {}, slots: {} }]
        }
      }
    }
  },
  watch: {
    pageReady (val) {
      if (val) {
        this.$set(this, 'modelTabs', [
          { value: 'locations', label: this.$t('home.locations.tab') },
          { value: 'equipment', label: this.$t('home.equipment.tab') },
          { value: 'properties', label: this.$t('home.properties.tab') }
        ])
        this.loadModel(this.page)
      }
    }
  },
  methods: {
    addWidget (component, widgetType, parentContext, slot) {
      if (!slot) slot = 'default'
      if (!component.slots) component.slots = {}
      if (!component.slots[slot]) component.slots[slot] = []
      if (widgetType) {
        component.slots[slot].push({
          component: widgetType,
          config: {
            title: 'New Tab',
            icon: 'f7:squares_below_rectangle'
          },
          slots: { default: [] }
        })
        this.forceUpdate()
      }
    },
    getWidgetDefinition (componentType) {
      return ConfigurableWidgets[componentType] ? ConfigurableWidgets[componentType]() : null
    },
    ensureCardComponentExists (card) {
      if (!this.page.slots[this.currentModelTab][0].slots[card.key]) {
        this.page.slots[this.currentModelTab][0].slots[card.key] = [
          {
            component: (this.currentModelTab) === 'locations' ? 'oh-location-card' : (this.currentModelTab === 'equipment') ? 'oh-equipment-card' : 'oh-property-card',
            config: {}
          }
        ]
      }
    },
    configureCard (card) {
      if (!card.key) return
      if (!this.page.slots[this.currentModelTab] || !this.page.slots[this.currentModelTab][0] || !this.page.slots[this.currentModelTab][0].slots) return
      this.ensureCardComponentExists(card)
      return this.configureWidget(this.page.slots[this.currentModelTab][0].slots[card.key][0])
    },
    editCardCode (card) {
      if (!card.key) return
      if (!this.page.slots[this.currentModelTab] || !this.page.slots[this.currentModelTab][0] || !this.page.slots[this.currentModelTab][0].slots) return
      this.ensureCardComponentExists(card)
      return this.editWidgetCode(this.page.slots[this.currentModelTab][0].slots[card.key][0])
    },
    addCardSeparator (idx) {
      const orderedCards = this.cardGroups(this.currentModelTab, this.page).flat().map((e) => (e.separator) ? e : e.key)
      orderedCards.splice(idx, 0, { separator: 'New Section' })
      this.$set(this.page.slots[this.currentModelTab][0].config, 'cardOrder', orderedCards)
      this.renameCardSeparator(idx)
    },
    renameCardSeparator (idx) {
      const orderedCards = this.cardGroups(this.currentModelTab, this.page).flat().map((e) => (e.separator) ? e : e.key)
      if (orderedCards[idx].separator) {
        this.$f7.dialog.prompt('Enter the title of the separator:', null,
          (title) => {
            orderedCards[idx].separator = title
            this.$set(this.page.slots[this.currentModelTab][0].config, 'cardOrder', orderedCards)
          },
          null,
          orderedCards[idx].separator)
      }
    },
    removeCardSeparator (idx) {
      const orderedCards = this.cardGroups(this.currentModelTab, this.page).flat().map((e) => (e.separator) ? e : e.key)
      orderedCards.splice(idx, 1)
      this.$set(this.page.slots[this.currentModelTab][0].config, 'cardOrder', orderedCards)
    },
    cardClicked (ev, card, idx) {
      ev.cancelBubble = true
      let el = ev.target
      if (el.classList.contains('icon-checkbox')) {
        this.toggleCardDisplay(card)
        return
      }
      while (!el.classList.contains('media-item')) {
        if (el && el.classList.contains('menu')) return
        el = el.parentElement
      }
      if (card.separator) {
        this.renameCardSeparator(idx)
      }
      this.configureCard(card)
    },
    reorderCard (ev) {
      const orderedCards = this.cardGroups(this.currentModelTab, this.page).flat().map((e) => (e.separator) ? e : e.key)
      const newOrder = [...orderedCards]
      newOrder.splice(ev.to, 0, newOrder.splice(ev.from, 1)[0])
      this.$set(this.page.slots[this.currentModelTab][0].config, 'cardOrder', newOrder)
      this.cardListId = null
      this.showCardControls = false
      this.$nextTick(() => {
        this.cardListId = this.$f7.utils.id()
      })
    },
    isCardExcluded (card) {
      if (!card.key) return
      const page = this.page
      const type = this.currentModelTab
      const excludedCards = (page && page.slots && page.slots[type] && page.slots[type][0] && page.slots[type][0].config && page.slots[type][0].config.excludedCards) ? page.slots[type][0].config.excludedCards : []
      const excludedIdx = excludedCards.indexOf(card.key)
      return excludedIdx >= 0
    },
    toggleCardDisplay (card) {
      if (!card.key) return
      const page = this.page
      const type = this.currentModelTab
      const excludedCards = (page && page.slots && page.slots[type] && page.slots[type][0] && page.slots[type][0].config && page.slots[type][0].config.excludedCards) ? page.slots[type][0].config.excludedCards : []
      const excludedIdx = excludedCards.indexOf(card.key)
      if (excludedIdx < 0) {
        this.$set(this.page.slots[type][0].config, 'excludedCards', [...excludedCards, card.key])
      } else {
        this.page.slots[type][0].config.excludedCards.splice(excludedIdx, 1)
      }
    },
    toYaml () {
      this.pageYaml = YAML.stringify(Object.assign({ config: this.page.config }, this.page.slots))
    },
    fromYaml () {
      try {
        const updatedTabs = YAML.parse(this.pageYaml)
        this.$set(this.page, 'slots', updatedTabs)
        this.$set(this.page, 'config', this.page.slots.config)
        delete this.page.slots.config
        this.forceUpdate()
        return true
      } catch (e) {
        this.$f7.dialog.alert(e).open()
        return false
      }
    }
  }
}
</script>
