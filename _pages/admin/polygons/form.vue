<template>
  <div id="pageId" class="q-layout-page layout-padding">

    <div class="relative-position q-mb-lg backend-page">
      <div class="box" v-if="success">

        <div class="col-12">
          <locales v-model="locale" ref="localeComponent" :form="$refs.formContent"/>
        </div>

        <!--Form-->
        <q-form autocorrect="off" autocomplete="off" ref="formContent" class="full-width q-my-sm" v-if="locale.success"
                @submit="itemId?updateItem():createItem()" @validation-error="$alert.error($tr('isite.cms.message.formInvalid'))">
          <div class="row q-col-gutter-md">
            <div class="col-12">
              <q-input outlined dense v-model="locale.formTemplate.name"
                       :label="`${$tr('isite.cms.form.name')} (${locale.language})*`"
                       :rules="[val => !!val || $tr('isite.cms.message.fieldRequired')]"/>
              <div class="input-title">{{ `${$tr('isite.cms.form.description')} (${locale.language})*` }}</div>
              <q-field borderless v-model="locale.formTemplate.description" :rules="[val => !!val || $tr('isite.cms.message.fieldRequired')]">
                <q-editor class="full-width" v-model="locale.formTemplate.description"/>
              </q-field>
            </div>
            <div class="col-12">
              <l-map :draw-control="true" id="lMap" :zoom="mapZoom" :center="center" @draw:created="getPointValues"
                     :style="`height : 300px`" ref="map">
                <l-tile-layer name="OpenStreetMap" layer-type="base" :token="token"
                              attribution='&copy; <a target="_blank" href="http://osm.org/copyright">OpenStreetMap</a> contributors'
                              url="https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png" />
                <polygon-drawer />
                <l-polygon :lat-lngs="locale.formTemplate.points" />
              </l-map>
            </div>
          </div>
          <q-page-sticky
                  position="bottom-right"
                  :offset="[18, 18]">
            <!--Update button-->
            <q-btn v-if="$route.params.id" color="green" :loading="loading"
                   icon="fas fa-edit" :label="$tr('isite.cms.label.update')" type="submit"/>
            <!--Save button-->
            <q-btn v-else color="green" :loading="loading" icon="fas fa-edit"
                   :label="$tr('isite.cms.label.create')" type="submit"/>
          </q-page-sticky>
        </q-form>
      </div>
      <inner-loading :visible="loading"/>
    </div>
  </div>
</template>

<script>
  import {latLng, Icon} from "leaflet";
  import { LMap, LTileLayer, LLayerGroup, LPolygon } from 'vue2-leaflet';
  import 'leaflet/dist/leaflet.css';
  import polygonDrawer from '@imagina/qlocations/_components/polygonDrawer'
  import {mapGeolocationActions, mapGeolocationGetters} from "quasar-app-extension-geolocation/src/store";
  export default {
    components: {
      LMap,
      LTileLayer,
      LLayerGroup,
      LPolygon,
      polygonDrawer,
    },
    watch: {
      $route(to, from) {
        this.initForm()
      }
    },
    mounted() {
      this.$nextTick(function () {
        this.initForm()
        this.$root.$on('page.data.refresh', () => this.initForm())//Listen refresh event
      })
    },
    data() {
      return {
        locale: {},
        map:{
          class: null,
          polygon: null,
          drawing: null
        },
        center: false,
        mapZoom: 14,
        loading: false,
        success: false,
        itemId: false,
      }
    },
    props:{
      id:{
        default: null
      }
    },
    computed: {
      dataLocale() {
        return {
          fields: {
            points: [],
            options: {},
          },
          fieldsTranslatable: {
            name: '',
            description: '',
          }
        }
      },
      ...mapGeolocationGetters([
        'isPermissionKnown',
        'isPermissionGranted',
        'isPermissionPrompt',
        'isPermissionDenied',
        'hasPosition',
        'coords',
      ]),
      token(){
        this.$store.getters['qsiteApp/getSettingValueByName']('isite::api-maps')
      }
    },
    methods: {
      getPointValues(e){
        console.warn(e)
        this.locale.form.points = e.layer.editing.latlngs
      },
      async initForm() {
        this.loading = true
        this.success = false
        this.locale = this.$clone(this.dataLocale)
        this.itemId = this.id !==null?this.id:this.$route.params.id
        if (this.locale.success) this.$refs.localeComponent.vReset()
        await this.getLocationPermisssions()
        this.center = this.coords ? latLng(this.coords.latitude, this.coords.longitude) : ['4.642129714308486', '-74.11376953125001']
        await this.getData()
        this.success = true
        this.loading = false
      },
      getData() {
        return new Promise((resolve, reject) => {
          const itemId = this.$clone(this.itemId)
          if (itemId) {
            let configName = 'apiRoutes.qlocations.polygons'
            //Params
            let params = {
              refresh: true,
              params: {
                filter: {allTranslations: true}
              }
            }
            //Request
            this.$crud.show(configName, itemId, params).then(response => {
              this.orderDataItemToLocale(response.data)
              resolve(true)//Resolve
            }).catch(error => {
              this.$alert.error({message: this.$tr('isite.cms.message.errorRequest'), pos: 'bottom'})
              this.loading = false
              reject(false)//Resolve
            })
          } else {
            resolve(true)//Resolve
          }
        })
      },
      orderDataItemToLocale(data) {
        let orderData = this.$clone(data)
        this.locale.form = this.$clone(orderData)
        let pts = this.$clone(this.locale.form.points)
        pts = pts.flat(3)
        this.center = pts[pts.length - 1]
      },
      async updateItem() {
        if (await this.$refs.localeComponent.validateForm()) {
          this.loading = true
          let configName = 'apiRoutes.qlocations.polygons'
          this.$crud.update(configName, this.itemId, this.getDataForm()).then(response => {
            this.$alert.success({message: `${this.$tr('isite.cms.message.recordUpdated')}`})
            //this.initForm()
            this.loading = false
          }).catch(error => {
            this.loading = false
            this.$alert.error({message: this.$tr('isite.cms.message.recordNoUpdated'), pos: 'bottom'})
          })
        }
      },
      async createItem() {
        if (await this.$refs.localeComponent.validateForm()) {
          this.loading = true
          let configName = 'apiRoutes.qlocations.polygons'
          this.$crud.create(configName, this.getDataForm()).then(response => {
            this.$alert.success({message: `${this.$tr('isite.cms.message.recordCreated')}`})
            this.$router.push({name: 'qlocations.admin.polygons.index'})
            //this.initForm()
            this.loading = false
          }).catch(error => {
            this.loading = false
            this.$alert.error({message: this.$tr('isite.cms.message.recordNoUpdated'), pos: 'bottom'})
          })
        }
      },
      getDataForm() {
        let response = this.locale.form
        for (var item in response) {
          let valueItem = response[item]
          if (valueItem == null || valueItem == undefined)
            delete response[item]
        }
        //response.selectable = JSON.stringify(response.selectable)
        return response
      },
      doQueryPermission () {
        this.queryPermission()
            .then(() => {
              if (this.isPermissionDenied) {
                // poll permission as the user might allow them in a separate tab
                this.pollingTimer = setTimeout(() => this.doQueryPermission(), 2000)
              } else if (this.pollingTimer) {
                clearTimeout(this.pollingTimer)
              }
            })
      },
      getLocationPermisssions () {
        this.samplePosition()
            .catch(() => { })
            .finally(() => {
              // update permissions (as the user might have enabled them)
              this.doQueryPermission()
            })
      },
      ...mapGeolocationActions([
        'samplePosition',
        'queryPermission'
      ])
    }
  }
</script>

<style scoped>

</style>
