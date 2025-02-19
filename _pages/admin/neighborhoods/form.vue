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
            <div class="col-12 col-md-6">
              <q-input outlined dense v-model="locale.formTemplate.name"
                       :label="`${$tr('isite.cms.form.name')} (${locale.language})*`"
                       :rules="[val => !!val || $tr('isite.cms.message.fieldRequired')]"/>
            </div>
            <div class="col-12 col-md-6">
              <!--Crud countries-->
              <crud
                  ref="countries"
                  :crud-data="import('@imagina/qlocations/_crud/countries')"
                  v-model="locale.formTemplate.countryId" type="select"
                  :crud-props="{label : `${$tr('ilocations.cms.form.country')} *`, clearable: true, rules: [val => !!val || $tr('isite.cms.message.fieldRequired')]}"
                  :config="{options : {label : 'name', value : 'id'}}"
                  @input="()=> { if(locale.formTemplate.countryId) $refs.provinces.init() }"
                  />
            </div>
            <div class="col-12 col-md-6">
              <!--Crud provinces-->
              <crud
                  ref="provinces"
                  :crud-data="import('@imagina/qlocations/_crud/provinces')"
                  v-model="locale.formTemplate.provinceId" type="select"
                  :crud-props="{label : `${$tr('ilocations.cms.form.province')} *`, clearable: true, rules: [val => !!val || $tr('isite.cms.message.fieldRequired')]}"
                  :config="{options : {label : 'name', value : 'id'}, requestParams: {filter: {country: locale.formTemplate.countryId}}}"
                  @input="()=> { if(locale.formTemplate.provinceId) $refs.cities.init() }"
                  />
            </div>
            <div class="col-12 col-md-6">
              <div class="row q-col-gutter-md">
                <div class="col-8 col-md-10">
                  <!--Crud cities-->
                  <crud
                      ref="cities"
                      :crud-data="import('@imagina/qlocations/_crud/cities')"
                      v-model="locale.formTemplate.cityId" type="select"
                      :crud-props="{label : `${$tr('ilocations.cms.form.city')} *`, clearable: true, rules: [val => !!val || $tr('isite.cms.message.fieldRequired')]}"
                      :custom-data="{create: false}"
                      :config="{options : {label : 'name', value : 'id'}, requestParams: {filter: {province_id: locale.formTemplate.provinceId}}}"
                  />
                </div>
                <div class="col-4 col-md-2 text-right">
                  <q-btn size="sm" class="btn-small" color="green" icon="fas fa-plus" @click="showCityCreate = true">
                    <q-tooltip :offset="[10, 10]">
                      {{ $tr('ilocations.cms.newCity') }}
                    </q-tooltip>
                  </q-btn>
                </div>
              </div>
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
    <q-dialog content-class="modal-form-crud"
              v-model="showCityCreate" v-if="showCityCreate" @hide="getCities" no-esc-dismiss no-backdrop-dismiss>
      <q-card class="bg-grey-1 backend-page row" style="max-width: 80vw!important;">
        <!--Header-->
        <q-toolbar class="bg-primary text-white">
          <q-toolbar-title>
            <label>{{$tr('ilocations.cms.newCity')}}</label>
          </q-toolbar-title>
          <q-btn flat v-close-popup icon="fas fa-times"/>
        </q-toolbar>
        <!--Content-->
        <q-card-section class="relative-position col-12">
          <cityForm :id="false" />
        </q-card-section>
      </q-card>
    </q-dialog>
  </div>
</template>

<script>
  import cityForm from '@imagina/qlocations/_pages/admin/cities/form'
  export default {
    components: {
      cityForm
    },
    watch: {
      $route(to, from) {
        this.initForm()
      }
    },
    mounted() {
      this.$nextTick(function () {
        this.initForm()
      })
    },
    data() {
      return {
        locale: {},
        loading: false,
        success: false,
        itemId: false,
        showCityCreate: false,
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
            code: '',
            countryId: null,
            provinceId: null,
            cityId: null,
          },
          fieldsTranslatable: {
            name: '',
          }
        }
      },
    },
    methods: {
      async initForm() {
        this.loading = true
        this.success = false
        this.locale = this.$clone(this.dataLocale)
        this.itemId = this.id !==null?this.id:this.$route.params.id
        if (this.locale.success) this.$refs.localeComponent.vReset()
        this.getCountries()
        await this.getData()
        this.success = true
        this.loading = false
      },
      getData() {
        return new Promise((resolve, reject) => {
          const itemId = this.$clone(this.itemId)
          if (itemId) {
            let configName = 'apiRoutes.qlocations.neighborhoods'
            //Params
            let params = {
              refresh: true,
              params: {
                include: 'city',
                filter: {allTranslations: true}
              }
            }
            //Request
            this.$crud.show(configName, itemId, params).then(response => {
              this.orderDataItemToLocale(response.data)
              this.locale.form.countryId = parseInt(response.data.city.countryId)
              this.locale.form.provinceId = parseInt(response.data.city.provinceId)
              this.locale.form.cityId = parseInt(response.data.cityId)
              this.getProvinces()
              this.getCities()
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
      },
      async updateItem() {
        if (await this.$refs.localeComponent.validateForm()) {
          this.loading = true
          let configName = 'apiRoutes.qlocations.neighborhoods'
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
          let configName = 'apiRoutes.qlocations.neighborhoods'
          this.$crud.create(configName, this.getDataForm()).then(response => {
            this.$alert.success({message: `${this.$tr('isite.cms.message.recordCreated')}`})
            this.$router.push({name: 'qlocations.admin.neighborhoods.index'})
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
