<template>
  <div class="searchblock">
    <v-container wrap class="">
      <v-row>
        <v-col sm="6" xs="12">
          <v-autocomplete
            v-model="model"
            :items="items"
            :loading="isLoading"
            :search-input.sync="search"
            hide-no-data
            hide-selected
            hide-details
            solo
            item-text="label"
            item-value="API"
            label="Search the Map"
            placeholder="Ex: 600 E 4th St"
            :filter="goFilter"
            return-object
            clearable
            aria-label="search"
            append-icon=""
          ></v-autocomplete>
        </v-col>
        <v-col sm="6" id="selectGroup">
          <v-select v-if="groups.length > 0"
            @change="selectGroup"
            :items="groups"
            label="Geography (approximate)"
            hide-details
            clearable
            solo
            aria-label="select geography group"
          ></v-select>
        </v-col>
      </v-row>
    </v-container>
  </div>
</template>

<script>
  import { mdiMapSearch } from '@mdi/js'

  export default {
    props: {
      value: Boolean,
      metricId: String,
      geometry: Object,
      mdiMapSearch: mdiMapSearch
    },
    data() {
      return {
        groups: [],
        descriptionLimit: 60,
        entries: [],
        isLoading: false,
        model: null,
        search: null,
      }
    },
    computed: {
      dataOptions() {
        return this.$store.state.dataOptions
      },
      geojson() {
        return this.$store.getters.geojsonName(this.metricId)
      },
      geojsonOptions() {
        return this.dataOptions.geojson.filter(elem => { return elem.layer === this.geojson })[0]
      },
      items () {
        return this.entries
      }
    },
    mounted () {
      if (this.geojsonOptions.selectGroups) {
        this.groups = this.geojsonOptions.selectGroups.map(elem => { return elem.name })
      }
    },
    watch: {
      search (newValue, oldValue) {
        if (!newValue || newValue === oldValue || this.isLoading) return

        this.isLoading = true

        // geometry id matches
        let geomMatches = this.geometry.features.filter(el => { return el.properties.id.toUpperCase().indexOf(newValue.toUpperCase()) !== -1 })
        this.entries = geomMatches.map(el => { return { label: `${this.geojsonOptions.name} ${el.properties.id}`, id: [el.properties.id] } })



        if (newValue.length >= 4) {
          // build urls
          const urls = []
          const searchPaths = this.dataOptions.searchPaths.concat(this.geojsonOptions.searchPaths)

          searchPaths.forEach(search => {
            let val = search.searchVal(newValue) || newValue
            urls.push(search.url + val)
          })

          const promises = urls.map(url => fetch(url).then(y => y.json()))
          Promise.all(promises)
            .then(results => {
              results.forEach((result, idx) => {
                let final = result
                if (searchPaths[idx].format) {
                  final = searchPaths[idx].format(result)
                }
                this.entries.push(...final)
              })
            })
            .then(e => {
              this.isLoading = false
            })
            .catch(error => {
              this.isLoading = false
              console.log(error)
            })
        } else {
          this.isLoading = false
        }

      },
      model(val) {
        if (val) {
          if (val.id) {
            this.$store.commit("addSelected", { geography: this.geojson, id: val.id })
            this.$emit('geocode', {id: val.id})
          }
          if (val.lnglat) {
            this.$emit('geocode', val)
          }
        }
      }
    },
    methods: {
      selectGroup(value) {
        if (!value) return false
        const group = this.geojsonOptions.selectGroups.filter(elem => { return elem.name === value })[0]
        fetch(group.ids)
          .then((resp) => resp.json()) // Transform the data into json
          .then((data) => {
            if (group.filter) data = group.filter(data)
            this.$store.commit("setSelected", { geography: this.geojson, selected: data })
            this.$emit('geocode', {id: data})
          })
      },
      goFilter() {
        return true
      }
    }
  }
</script>

<style scoped>
.v-text-field {
  padding-top: 0;
  margin-top: 0;
}

/* .searchblock {
  background-color: #ccc;
} */

@media (max-width: 600px) {
  #selectGroup {
    display: none;
  }
}
</style>