<template>
  <div id="pageId" class="q-layout-page layout-padding location-form">

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
                       :label="`${$tr('isite.cms.form.name')}*`"
                       :rules="[val => !!val || $tr('isite.cms.message.fieldRequired')]"/>
              <div class="input-title">{{ `${$tr('isite.cms.form.description')}*` }}</div>
              <q-field borderless v-model="locale.formTemplate.description" :rules="[val => !!val || $tr('isite.cms.message.fieldRequired')]">
                <q-editor class="full-width" v-model="locale.formTemplate.description"/>
              </q-field>
            </div>
          </div>
          <div class="q-py-sm">
            <div class="text-right">
              <q-btn color="positive" icon="fas fa-plus" @click="addGeozonables" />
            </div>
            <div v-for="(item, index) in locale.form.zonesToGeozone">
                <div class="row q-col-gutter-sm">
                  <div class="col-3">
                    <dynamic-field
                        v-model="item.countryId"
                        :field="{
                          value: '',
                          type: 'select',
                          loadOptions: {
                            apiRoute: 'apiRoutes.qlocations.countries',
                            select: {label: 'name', id: 'id'},
                          },
                          props : {
                            clearable: true,
                            label: `${$tr('ilocations.cms.form.country')}*`,
                            rules: [val => !!val || $tr('isite.cms.message.fieldRequired')]
                          }
                        }"
                    />
                  </div>
                  <div class="col-3">
                    <dynamic-field
                        v-if="item.countryId > 0"
                        v-model="item.provinceId"
                        :field="{
                          value: '0',
                          type: 'select',
                          loadOptions: {
                            apiRoute: 'apiRoutes.qlocations.provinces',
                            requestParams: {
                             filter: {
                               country: item.countryId
                             }
                            },
                            select: {label: 'name', id: 'id'},
                          },
                          props : {
                            clearable: true,
                            label: `${$tr('ilocations.cms.form.province')}*`,
                            options:[{
                              label: $trp('isite.cms.label.all'), value: '0'
                            }],
                          }
                        }"
                    />
                  </div>
                  <div class="col-4">
                    <dynamic-field
                        v-if="item.provinceId > 0"
                        v-model="item.cityId"
                        :field="{
                          value: '0',
                          type: 'select',
                          loadOptions: {
                            apiRoute: 'apiRoutes.qlocations.cities',
                            requestParams: {
                             filter: {
                               provinceId: item.provinceId
                             }
                            },
                            select: {label: 'name', id: 'id'},
                          },
                          props : {
                            clearable: true,
                            label: `${$tr('ilocations.cms.form.city')}*`,
                            options:[{
                              label: $trp('isite.cms.label.all'), value: '0'
                            }],
                          }
                        }"
                    />
                  </div>
                  <div class="col-2">
                    <q-btn color="red" icon="fas fa-trash" @click="deleteGeozonable(index)" />
                  </div>
                </div>
            </div>
          </div>
          <q-page-sticky
                  position="bottom-right"
                  :offset="[18, 18]">
            <!--Update button-->
            <q-btn v-if="$route.params.id" color="positive" :loading="loading"
                   icon="fas fa-edit" :label="$tr('isite.cms.label.update')" type="submit"/>
            <!--Save button-->
            <q-btn v-else color="positive" :loading="loading" icon="fas fa-edit"
                   :label="$tr('isite.cms.label.create')" type="submit"/>
          </q-page-sticky>
        </q-form>
      </div>
      <inner-loading :visible="loading"/>
    </div>
  </div>
</template>

<script>
  import cityForm from '@imagina/qlocations/_pages/admin/cities/form'
  import polygonForm from '@imagina/qlocations/_pages/admin/polygons/form'
  import neighborhoodForm from '@imagina/qlocations/_pages/admin/neighborhoods/form'
  export default {
    components: {
      cityForm,
      polygonForm,
      neighborhoodForm,
    },
    watch: {
      $route(to, from) {
        this.initForm()
      }
    },
    mounted() {
      this.$nextTick(function () {
        this.initForm()
        this.$root.$on('page.data.refresh', () => this.initForm())
      })
    },
    data() {
      return {
        locale: {},
        loading: false,
        success: false,
        itemId: false,
        showCityCreate: false,
        showPolygonCreate: false,
        showNeighborhoodCreate: false,
        geozonables:[]
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
            name: '',
            description: '',
            countries: [],
            provinces: [],
            cities: [],
            polygons: [],
            neighborhoods: [],
            zonesToGeozone: []
          },
          fieldsTranslatable: {
          }
        }
      },
      defaultGeozonable(){
        return {
          countryId: '',
          provinceId: '0',
          cityId: '0',
        }
      }
    },
    methods: {
      async initForm() {
        this.loading = true
        this.success = false
        this.locale = this.$clone(this.dataLocale)
        this.itemId = this.id !==null?this.id:this.$route.params.id
        if (this.locale.success) this.$refs.localeComponent.vReset()
        await this.getData()
        this.success = true
        this.loading = false
      },
      getData() {
        return new Promise((resolve, reject) => {
          const itemId = this.$clone(this.itemId)
          if (itemId) {
            let configName = 'apiRoutes.qlocations.geozones'
            //Params
            let params = {
              refresh: true,
              params: {
                include: 'zonesToGeozone,zonesToGeozone.province,zonesToGeozone.country,zonesToGeozone.city',
                filter: {allTranslations: true}
              }
            }
            //Request
            this.$crud.show(configName, itemId, params).then(response => {
              this.orderDataItemToLocale(response.data)
              let geozones = this.$clone(response.data.zonesToGeozone)
              /*this.locale.form.neighborhoods = response.data.neighborhoods.map(item =>{
                return item.id
              })
              this.locale.form.polygons = response.data.polygons.map(item =>{
                return item.id
              })*/
              resolve(true)//Resolve
            }).catch(error => {
              console.error(error)
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
      },
      async updateItem() {
        if (await this.$refs.localeComponent.validateForm()) {
          this.loading = true
          let configName = 'apiRoutes.qlocations.geozones'
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
          let configName = 'apiRoutes.qlocations.geozones'
          this.$crud.create(configName, this.getDataForm()).then(response => {
            this.$alert.success({message: `${this.$tr('isite.cms.message.recordCreated')}`})
            this.$router.push({name: 'qlocations.admin.geozones.index'})
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
        response.selectable = JSON.stringify(response.selectable)
        return response
      },
      addGeozonables(){
        this.locale.form.zonesToGeozone.push(this.defaultGeozonable)
      },
      deleteGeozonable(index){
        this.locale.form.zonesToGeozone.splice(index,1)
      },
    }
  }
</script>

<style lang="stylus">
  .location-form
    .modal-form-crud
      .layout-padding
        padding 0!important
    #crudContentPage
      .q-btn
        .q-btn__content
          div
            display none
</style>
