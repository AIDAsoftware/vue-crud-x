<script>
import _cloneDeep from 'lodash.clonedeep'
const CrudStore = {
  namespaced: true,
  // strict: true,
  state: {
    records: [], // get many - filter, page & sort
    totalRecs: 0,
    record: { }, // selected record
    pagination: { },
    filterData: { },
    defaultRec: { },
    crudOps: {
      export: null,
      find: null,
      delete: null,
      findOne: null,
      create: null,
      update: null
    }
  },
  getters: {
    record (state) { return state.record },
    records (state) { return state.records },
    totalRecs (state) { return state.totalRecs },
    filterData (state) { return state.filterData },
    pagination (state) { return state.pagination },
    defaultRec (state) { return state.defaultRec },
    crudOps (state) { return state.crudOps }
  },
  mutations: {
    setRecords (state, payload) {
      state.records = payload.records
      state.totalRecs = payload.totalRecs
    },
    setRecord (state, payload) {
      if (payload === null) state.record = (typeof state.defaultRec === 'function') ? state.defaultRec() : _cloneDeep(state.defaultRec)
      else state.record = _cloneDeep(payload)
    },
    setPagination (state, payload) { state.pagination = payload },
    setFilterData (state, payload) { state.filterData = payload }
  },
  actions: { // Edit Actions
    setPagination ({commit}, payload) {
      commit('setPagination', payload)
    },
    async deleteRecord ({commit, getters}, payload) {
      payload.user = this.getters.user
      let res = await getters.crudOps.delete(payload)
      return res
    },
    async getRecord ({commit, getters}, payload) {
      payload.user = this.getters.user
      let record = await getters.crudOps.findOne(payload)
      commit('setRecord', record)
    },
    async getRecords ({commit, getters}, payload) {
      payload.user = this.getters.user
      let {records, pagination} = await getters.crudOps.find(payload)
      let totalRecs = payload.doPage ? pagination.totalItems : records.length
      commit('setPagination', pagination)
      commit('setFilterData', payload.filterData)
      commit('setRecords', {records, totalRecs})
    },
    async exportRecords ({commit, getters}, payload) {
      payload.user = this.getters.user
      await getters.crudOps.export(payload)
    },
    async updateRecord ({commit, getters}, payload) {
      payload.user = this.getters.user
      let res = await getters.crudOps.update(payload)
      return res
    },
    async createRecord ({commit, getters, dispatch}, payload) {
      payload.user = this.getters.user
      let res = await getters.crudOps.create(payload)
      return res
    },
    resetRecord ({ commit }) {
      commit('setRecord', null)
    }
  }
}
export default {
  name: 'VueCrudX',
  props: {
    parentId: { type: String, default: null },
    storeName: { type: String, required: true },
    crudFilter: { type: Object, required: true },
    crudTable: { type: Object, required: true },
    crudForm: { type: Object, required: true },
    crudOps: { type: Object, required: true },
    withHeader: {type: Boolean, default: true},
    crudTitle: { type: String },
    doPage: { type: Boolean, default: true },
    crudSnackBar: { type: Object, default: () => ({ bottom: true, timeout: 6000 }) },
    fixed: { type: Boolean, default: false },
  },
  created () {
    const store = this.$store
    const name = this.storeName
    if (!(store && store.state && store.state[name])) { // register a new module only if doesn't exist
      store.registerModule(name, _cloneDeep(CrudStore)) // make sure its a deep clone
      store.state[name].defaultRec = this.crudForm.defaultRec
      store.state[name].filterData = this.crudFilter.filterData
      store.state[name].crudOps = this.crudOps
    } else { // re-use the already existing module
    }
    this.$options.filters.formatters = this.crudTable.formatters // create the formatters programatically
    this.headers = this.crudTable.headers.map(header => Object.assign({}, header, {text: this.$translate(header.text)})) 
    this.inline = this.crudTable.inline || false
    this.confirmCreate = this.crudTable.confirmCreate || false
    this.confirmUpdate = this.crudTable.confirmUpdate || false
    this.confirmDelete = this.crudTable.confirmDelete || false
    this.$options.components['crud-filter'] = this.crudFilter.FilterVue
    this.$options.components['crud-form'] = this.crudForm.FormVue
    if (this.record.id && !this.parentId) { // nested CRUD
      this.addEditDialogFlag = true
    }
  },
  beforeUpdate () { },
  mounted () { },
  beforeRouteEnter (to, from, next) { next(vm => { }) },
  data () {
    return {
      // form
      addEditDialogFlag: false,
      validForm: true,
      // filter
      validFilter: true,
      // data-table
      loading: false,
      headers: { }, // pass in
      inline: false, // inline editing
      inlineValue: null,
      confirmCreate: false, // confirmation required flags
      confirmUpdate: false,
      confirmDelete: true,
      // snackbar
      snackbar: false,
      snackbarText: ''
    }
  },
   destroyed: function () {
      this.resetRecord()
      this.addEditDialogFlag = false
  },
  computed: {
    showTitle () { return this.$translate(this.crudTitle || this.storeName) }, 
    // ...mapGetters(storeModuleName, [ 'records', 'totalRecs', 'filterData', 'record' ]), // cannot use for multiple stores, try below
    records () { return this.$store.getters[this.storeName + '/records'] },
    totalRecs () { return this.$store.getters[this.storeName + '/totalRecs'] },
    filterData () { return this.$store.getters[this.storeName + '/filterData'] },
    // pagination () { return this.$store.getters[this.storeName + '/pagination'] }, // not used
    record () { return this.$store.getters[this.storeName + '/record'] },
    pagination: {
      get: function () {
        let rv = { }
        try {
          rv = this.$store.state[this.storeName].pagination
        } catch (e) {
          // console.log('Catch computed pagination:', e.message)
        }
        return rv
      },
      set: function (value) {
        this.setPagination(value)
      }
    },
    hasFilterForm () {
      return !this.crudFilter.FilterVue()
    }
  },
  filters: {
    capitalize: function (value) {
      if (!value) return ''
      value = value.toString()
      return value.charAt(0).toUpperCase() + value.slice(1)
    }
  },
  watch: {
    loading: function (newValue, oldValue) { },
    pagination: {
      handler () {
        this.getRecordsHelper()
      },
      deep: true
    }
  },
  methods: {
    setSnackBar (statusCode) {
      if (this.crudSnackBar && statusCode) {
        this.snackbarText = 'Unknown Operation'
        if (statusCode === 200 || statusCode === 201) this.snackbarText = 'OK'
        else if (statusCode === 500) this.snackbarText = 'Operation Error'
        else if (statusCode === 409) this.snackbarText = 'Duplicate Error'
        this.snackbar = true
      }
    },
    async getRecords (payload) { await this.$store.dispatch(this.storeName + '/getRecords', payload) },
    setPagination (payload) { this.$store.dispatch(this.storeName + '/setPagination', payload) },
    async deleteRecord (payload) {
      this.emitLoading(true)
      let res = await this.$store.dispatch(this.storeName + '/deleteRecord', payload)
      this.emitLoading(false)
      this.setSnackBar(res)
      return res === 200
    },
    async updateRecord (payload) {
      this.emitLoading(true)
      let res = await this.$store.dispatch(this.storeName + '/updateRecord', payload)
      this.emitLoading(false)
      this.setSnackBar(res)
      return res
    },
    async createRecord (payload) {
      this.emitLoading(true)
      let res = await this.$store.dispatch(this.storeName + '/createRecord', payload)
      this.emitLoading(false)
      this.setSnackBar(res)
      return res
    },
    async getRecord (payload) {
      this.emitLoading(true)
      await this.$store.dispatch(this.storeName + '/getRecord', payload)
      this.emitLoading(false)
    },
    setRecord (payload) { this.$store.commit(this.storeName + '/setRecord', null) },
    resetRecord () { this.$store.dispatch(this.storeName + '/resetRecord') },
    async exportRecords (payload) { await this.$store.dispatch(this.storeName + '/exportRecords', payload) },
    closeAddEditDialog () {
      this.resetRecord()
      this.resetForm()
      this.addEditDialogFlag = false
    },
    async addEditDialogOpen (id) {
      if (id) await this.getRecord({id}) // edit
      else this.setRecord() // add
      this.addEditDialogFlag = true
    },
    async addEditDialogSave (e) {
      if (this.record.id && this.confirmCreate) if (!confirm(this.getConfirmText())) return
      if (!this.record.id && this.confirmUpdate) if (!confirm(this.getConfirmText())) return

      let closeDialog = true 
      if (this.record.id) closeDialog = await this.updateRecord({record: this.record})
      else closeDialog = await this.createRecord({record: this.record, parentId: this.parentId})
      if (closeDialog) { 
        await this.getRecordsHelper()
        this.closeAddEditDialog()
      }
    },
    async addEditDialogDelete (e) {
      if (this.confirmDelete) if (!confirm(this.getConfirmText())) return
      const {id} = this.record
      if (id) {
        await this.deleteRecord({id})
        await this.getRecordsHelper()
      }
      this.closeAddEditDialog()
    },
    async getRecordsHelper () {
      this.loading = true
      await this.getRecords({
        doPage: this.doPage,
        pagination: this.pagination,
        filterData: this.filterData,
        parentId: this.parentId
      })
      this.loading = false
    },
    async submitFilter () {
      await this.getRecordsHelper()
    },
    async exportBtnClick () {
      this.loading = true
      await this.exportRecords({
        pagination: this.pagination,
        filterData: this.filterData,
        parentId: this.parentId
      })
      this.loading = false
    },
    // clearFilter () { // can do test code here too
    //   this.$refs.searchForm.reset()
    // },
    goBack () { this.$router.back() },
    // inline
    inlineOpen (value) {
      this.inlineValue = value
    },
    async inlineUpdate (item, field) {
      const rv = await this.updateRecord({record: item})
      if (!rv) item[field] = this.inlineValue // if false undo changes
    },
    async inlineCreate () {
      if (this.confirmCreate) if (!confirm(this.getConfirmText())) return
      await this.createRecord({record: (typeof this.crudForm.defaultRec === 'function') ? this.crudForm.defaultRec() : this.crudForm.defaultRec, parentId: this.parentId})
      this.$nextTick(async function () { await this.getRecordsHelper() })
    },
    async inlineDelete (id) {
      if (this.confirmDelete) if (!confirm(this.getConfirmText())) return
      await this.deleteRecord({id})
      this.$nextTick(async function () { await this.getRecordsHelper() })
    },
    getConfirmText () { 
      return this.$t ? this.$t('vueCrudX.confirm') : this.$translate('vueCrudX.confirm')
    },
    submitForm () {
      if (this.$refs['submitForm'].validate()) { 
        this.addEditDialogSave()
      }
    },
    resetForm () {
      this.$refs['submitForm'].reset()
    },
    emitLoading (status) {
      this.loading = status      
      this.$emit("loading",status)
    }
  }
}
</script>

<template>
  <v-container v-bind:class="{ 'make-modal': parentId }"> 
    <v-layout row justify-end>
      <v-btn v-if="parentId" fab top @click.stop="goBack" :disabled="loading"><v-icon>reply</v-icon></v-btn>
      <v-btn class="new-button" v-if="crudOps.create" depressed color="secondary" @click.stop="inline?inlineCreate():addEditDialogOpen(null)" :disabled="loading">{{ this.$translate('Shared.New') }}</v-btn>
      <v-btn v-if="crudOps.export" fab top @click.stop="exportBtnClick" :disabled="loading"><!-- handle disabled FAB in Vuetify -->
        <v-icon :class='[{"white--text": !loading }]'>print</v-icon>
      </v-btn>
    </v-layout>
    <v-expansion-panel v-if="withHeader" :disabled="hasFilterForm">
      <v-expansion-panel-content class="primary">
        <div slot="header" ><span class="expandTitle">{{showTitle | capitalize}}</span></div>
        <v-form class="grey lighten-3 pa-2" v-model="validFilter" ref="searchForm" lazy-validation>
          <crud-filter :filterData="filterData" :parentId="parentId" :storeName="storeName" />
          <v-layout row justify-end>
            <!-- v-btn fab @click="clearFilter"><v-icon>close</v-icon></v-btn -->
            <v-btn fab @click="submitFilter" :disabled="!validFilter || loading"><v-icon>replay</v-icon></v-btn>
          </v-layout>
        </v-form>
      </v-expansion-panel-content>
    </v-expansion-panel>
    <v-data-table
      :headers="headers"
      :items="records"
      :total-items="totalRecs"
      :pagination.sync="pagination"
      :loading="loading"
      class="elevation-1"
      :hide-actions=!doPage
    >
      <template slot="items" slot-scope="props">
        <!-- tr @click.stop="(e) => addEditDialogOpen(e, props.item.id, $event)" AVOID ARROW fuctions -->
        <tr v-if="!inline" @click.stop="addEditDialogOpen(props.item.id)">
          <td class="pointer" :key="header.value" v-for="header in headers">{{ props.item[header.value] | formatters(header.value) }}</td>
        </tr>
        <tr v-else>
          <!-- for now, lighten (grey lighten-4) editable columns until fixed header is implemented -->
          <td :key="header.value" v-for="header in headers" :class="{ 'grey lighten-4': (inline[header.value] && crudOps.update) }">
            <v-edit-dialog v-if="inline[header.value] && crudOps.update"
              :return-value.sync="props.item[header.value]"
              large
              lazy
              persistent
              @save="inlineUpdate(props.item, header.value)"
              @cancel="()=>{}"
              @open="inlineOpen(props.item[header.value])"
              @close="()=>{}"
            >
              <div>{{ props.item[header.value] }}</div>
              <div slot="input" class="mt-3 title">Update Field</div>
              <v-text-field slot="input" v-model="props.item[header.value]" label="Edit" :type="inline[header.value]" single-line counter autofocus></v-text-field>
            </v-edit-dialog>
            <span v-else>{{ props.item[header.value] | formatters(header.value) }}</span>
          </td>
          <td>
            <v-btn v-if="crudOps.delete" @click.native="inlineDelete(props.item.id)"><v-icon>delete</v-icon></v-btn>
          </td>
        </tr>
      </template>
      <template slot="no-data">
        <v-flex class="text-xs-center">
          <v-icon>warning</v-icon> {{$t?$t('vueCrudX.noData'): this.$translate('vueCrudX.noData')}} 
        </v-flex>
      </template>
    </v-data-table>

    <v-layout row justify-center>
      <v-dialog v-model="addEditDialogFlag" fullscreen transition="none" :overlay="false">
        <v-card>
          <div class="with-padding-top">
            <div class="backToolbarContainer">
              <div class="backToolbarContent">
                <v-icon @click="closeAddEditDialog()" color="secondaryVariant" class="with-icon">arrow_back</v-icon><span class="onBackground--text">{{ this.$translate(crudTitle) }}</span>
              </div>
            </div>
            <v-card class="with-card">
              <v-form ref="submitForm" class="pa-2" v-model="validForm" lazy-validation>
              <crud-form :record="record" :parentId="parentId" :storeName="storeName"/>
                <v-card-actions class="with-space-between">
                  <v-btn v-if="(record.id && this.crudOps.update) || (!record.id && this.crudOps.create)" :disabled="!validForm || loading" color="secondary" @click.native="submitForm">{{ this.$translate("Shared.Done") }}</v-btn>
                  <v-btn v-if="record.id && crudOps.delete" depressed color="surfaceHover" @click.native="addEditDialogDelete" :disabled="loading" >{{ this.$translate("Shared.Delete") }}</v-btn>
                </v-card-actions>
              </v-form>
            </v-card>
          </div>
        </v-card>
      </v-dialog>
    </v-layout>
    <v-snackbar v-if="crudSnackBar" v-model="snackbar" v-bind="crudSnackBar">
      {{ snackbarText }}
      <v-btn fab flat @click="snackbar=false"><v-icon >close</v-icon></v-btn>
    </v-snackbar>
  </v-container>
</template>

<style lang="css" scoped>
  .make-modal {
    margin: 0;
    position: fixed;
    top: 0;
    left: 0;
    z-index: 100;
    padding: 0;
    min-width: 100%;
    min-height: 100%;
    background-color: #fff;
  }
  .pointer { 
    cursor: pointer;
    text-align: left;
  }
  .expandTitle { 
    color: white; 
  }
  .backToolbarContent {
    max-width: 60%;
    width: 60%;
    text-align: left;
  }
  .backToolbarContainer {
    display: flex;
    font-size: 16px;
    justify-content: center;
    margin-bottom: 20px;
  }
  .backToolbarContent span {
    margin-left: 10px;
  }
  .with-padding-top {
    padding-top: 125px;
  }
  .with-card {
    max-width: 60%;
    margin: auto;
  }
  .with-space-between {
    display: flex;
    flex-direction: row-reverse;
    justify-content: space-between;
    padding: 0 24px 24px 24px;
  }
  .new-button {
    margin-bottom: 10px;
  }
  .v-icon {
    line-height: 0.8;
  }
  .v-dialog--fullscreen {
    transition: 0s;
  }
</style>
