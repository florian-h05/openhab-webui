<template>
  <span class="padding-right location-status-badge" v-show="reduce" :class="{ invert: invertColor }">
    <oh-icon v-if="config.icon.indexOf('oh:') === 0" :icon="config.icon.replace('oh:', '')" :state="config.state" class="oh-icon-badge" width="20" height="20" />
    <f7-icon v-else-if="config.icon.indexOf('f7:') === 0" :f7="config.icon.replace('f7:', '')" :color="config.invertText ? 'black' : 'white'" class="f7-icon-badge" size="20" />
    <!-- <oh-icon v-if="config.icon.indexOf('oh:') === 0 && config.stateOff" v-show="!reduce" icon="config.icon.replace('oh:', '')"  :state="config.stateOff" class="oh-icon-badge" width="20" height="20" /> -->
    <span class="glance-label" v-show="reduce">{{reduce}} {{config.unit}}</span>
    <span class="glance-label" v-show="reduceAux"><small>({{reduceAux}} {{config.unit}})</small></span>
  </span>
</template>

<style lang="stylus">
.location-status-badge
  .oh-icon-badge
    filter brightness(100)
  &.invert .oh-icon-badge
    filter brightness(0)
  .f7-icon-badge
    line-height 20px
    margin-top -7px
  .glance-label
    line-height 20px
    vertical-align top
</style>

<script>
import { allEquipmentPoints, findPoints } from '../glance-helpers'

export default {
  props: ['element', 'type', 'customConfig', 'invertColor', 'store'],
  data () {
    return {
      badgeConfigs: {
        temperature: { icon: 'f7:thermometer', unit: '°' },
        humidity: { icon: 'f7:drop', unit: '%' },
        luminance: { icon: 'f7:sun_min', unit: 'lux' }
      }
    }
  },
  computed: {
    config () {
      return this.badgeConfigs[this.type]
    },
    query () {
      let direct
      switch (this.type) {
        case 'temperature':
          direct = findPoints(this.element.properties, 'Point_Measurement', true, 'Property_Temperature')
          if (direct.length) return direct
          return findPoints(allEquipmentPoints(this.element.equipment), 'Point_Measurement', true, 'Property_Temperature')
        case 'humidity':
          direct = findPoints(this.element.properties, 'Point_Measurement', true, 'Property_Humidity')
          if (direct.length) return direct
          return findPoints(allEquipmentPoints(this.element.equipment), 'Point_Measurement', true, 'Property_Humidity')
        case 'luminance':
          direct = findPoints(this.element.properties, 'Point_Measurement', true, 'Property_Light')
          if (direct.length) return direct
          return findPoints(allEquipmentPoints(this.element.equipment), 'Point_Measurement', true, 'Property_Light')
        default:
          return []
      }
    },
    queryAux () {
      let direct
      switch (this.type) {
        case 'temperature':
          direct = findPoints(this.element.properties, 'Point_Setpoint', true, 'Property_Temperature')
          if (direct.length) return direct
          return findPoints(allEquipmentPoints(this.element.equipment), 'Point_Setpoint', true, 'Property_Temperature')
        default:
          return []
      }
    },
    map () {
      return this.query.map((item) => this.store[item.name].state)
    },
    mapAux () {
      return this.queryAux.map((item) => this.store[item.name].state)
    },
    reduce () {
      const ret = this.map.reduce((avg, state, arr, { length }) => {
        const value = Number.parseFloat(state.split(' ')[0])
        if (Number.isFinite(value)) {
          return avg + value / length
        }
        return avg
      }, 0)

      return (this.type === 'temperature') ? Math.round(ret * 10) / 10 : Math.round(ret)
    },
    reduceAux () {
      if (this.type !== 'temperature') return undefined
      const ret = this.mapAux.reduce((avg, state, arr, { length }) => {
        const value = Number.parseFloat(state.split(' ')[0])
        if (Number.isFinite(value)) {
          return avg + value / length
        }
        return avg
      }, 0)

      return Math.round(ret * 10) / 10
    }
  }
}
</script>