<template></template>

<script>
  export default {
    data() {
      return {
        crudId: this.$uid()
      }
    },
    computed: {
      crudData() {
        return {
          crudId: this.crudId,
          entityName: config("main.qlocations.entityNames.city"),
          apiRoute: 'apiRoutes.qlocations.cities',
          //permission: 'icommerce.manufacturers',
          create: {
            title: this.$tr('ilocations.cms.newCity'),
            to: {name: 'qlocations.admin.cities.create'}
          },
          read: {
            columns: [
              {name: 'id', label: this.$tr('isite.cms.form.id'), field: 'id', style: 'width: 50px'},
              {name: 'name', label: this.$tr('isite.cms.form.name'), field: 'name', align: 'left'},
              {
                name: 'created_at', label: this.$tr('isite.cms.form.createdAt'), field: 'createdAt', align: 'left',
                format: val => val ? this.$trd(val) : '-',
              },
              {name: 'actions', label: this.$tr('isite.cms.form.actions'), align: 'left'},
            ],
            //requestParams: {include: 'parent'}
            filters:{
              country: {
                value: null,
                type: 'select',
                props : {
                  label: this.$tr('ilocations.cms.form.country'),
                },
                loadOptions: {
                  apiRoute: 'apiRoutes.qlocations.countries',
                  select: {label: 'name', id: 'id'},
                }
              },
              province: {
                value: null,
                type: 'select',
                props : {
                  label: this.$tr('ilocations.cms.form.province'),
                },
                loadOptions: {
                  apiRoute: 'apiRoutes.qlocations.provinces',
                  select: {label: 'name', id: 'id'},
                }
              },
            }
          },
          update: {
            title: this.$tr('ilocations.cms.updateCity'),
            to: 'qlocations.admin.cities.edit'
            //requestParams: {include: 'parent'}
          },
          delete: true,
          /*formLeft: {
          },
          /*formRight: {
          },*/
        }
      },
      //Crud info
      crudInfo() {
        return this.$store.state.qcrudComponent.component[this.crudId] || {}
      }
    }
  }
</script>
