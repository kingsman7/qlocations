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
              <q-input outlined dense v-model="locale.formTemplate.code"
                       :label="`${$tr('ilocations.cms.form.code')} (${locale.language})*`"
                       :rules="[val => !!val || $tr('isite.cms.message.fieldRequired')]"/>
            </div>
            <div class="col-12">
              <!--Crud countries-->
              <div class="row q-col-gutter-md">
                <div class="col-12 col-sm-6">
                  <crud
                      ref="countries"
                      :crud-data="import('@imagina/qlocations/_crud/countries')"
                      v-model="locale.formTemplate.countryId" type="select"
                      :crud-props="{label : `${$tr('ilocations.cms.form.country')} *`, clearable: true, rules: [val => !!val || $tr('isite.cms.message.fieldRequired')]}"
                      :config="{options : {label : 'name', value : 'id'}}"
                      @input="()=> { if(locale.formTemplate.countryId) $refs.provinces.init() }"
                  />
                </div>
                <div class="col-12 col-sm-6">
                  <crud
                      ref="provinces"
                      :crud-data="import('@imagina/qlocations/_crud/provinces')"
                      v-model="locale.formTemplate.provinceId" type="select"
                      :crud-props="{label : `${$tr('ilocations.cms.form.province')} *`, clearable: true, rules: [val => !!val || $tr('isite.cms.message.fieldRequired')]}"
                      :config="{options : {label : 'name', value : 'id'}, requestParams: {filter: {country: locale.formTemplate.countryId}}}"
                  />
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

  export default {
    components: {
    },
    watch: {
      $route(to, from) {
        this.initForm(true)
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
        countryLoading: false,
        provinceLoading: false,
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
            code: '',
            countryId: null,
            provinceId: null,
          },
          fieldsTranslatable: {
            name: '',
          }
        }
      },
    },
    methods: {
      async initForm(refresh = false) {
        this.loading = true
        this.success = false
        this.locale = this.$clone(this.dataLocale)
        this.itemId = this.id !==null?this.id:this.$route.params.id
        if (this.locale.success) this.$refs.localeComponent.vReset()
        await this.getData(refresh)
        this.success = true
        this.loading = false
      },
      getData(refresh = false) {
        return new Promise((resolve, reject) => {
          const itemId = this.$clone(this.itemId)
          if (itemId) {
            let configName = 'apiRoutes.qlocations.cities'
            //Params
            let params = {
              refresh: refresh,
              params: {
                include: 'province,country',
                filter: {allTranslations: true}
              }
            }
            //Request
            this.$crud.show(configName, itemId, params).then(response => {
              this.orderDataItemToLocale(response.data)
              //this.locale.form.countryId = parseInt(response.data.countryId)
              this.locale.form.provinceId = parseInt(response.data.provinceId)
              this.getProvinces()
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
          let configName = 'apiRoutes.qlocations.cities'
          this.$crud.update(configName, this.itemId, this.getDataForm()).then(response => {
            this.$alert.success({message: `${this.$tr('isite.cms.message.recordUpdated')}`})
            this.$router.push({name: 'qlocations.admin.cities.index'})
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
          let configName = 'apiRoutes.qlocations.cities'
          this.$crud.create(configName, this.getDataForm()).then(response => {
            this.$alert.success({message: `${this.$tr('isite.cms.message.recordCreated')}`})
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
    #crudContentPage
      .q-btn
        .q-btn__content
          div
            display none
</style>
