<template>
  <wrapper :loading="loading" :orient="orient">
    <template v-slot:goto>
      <router-link :to="`/dashboard/${dash.id}/logs`">switch to logs</router-link>
    </template>

    <template v-slot:filters>
      <form @change="onChangeFilters" @submit.prevent>
        <select v-model="filters.logname" id="filter-logname" :class="{ selected: filters.logname }">
          <option value="" v-if="sortedLognames.length === 0">Logname</option>
          <option v-for="logname in sortedLognames" :value="logname" :key="logname">
            {{ logname }}
          </option> </select
        ><select
          id="filter-hostname"
          v-model="filters.hostname"
          v-if="sortedHostnames.length > 1 || filters.hostname"
          :class="{ selected: filters.logname }"
        >
          <option value="">Hostname</option>
          <option v-for="hostname in sortedHostnames" :value="hostname" :key="hostname">
            {{ hostname }}
          </option> </select
        ><select v-model="filters.version" id="filter-version" :class="{ selected: filters.version }">
          <option value="">Version</option>
          <option v-for="version in sortedVersions" :value="version" :key="version" v-if="version">
            {{ version }}
          </option> </select
        ><select v-model="filters.agg" id="filter-agg" :class="{ selected: filters.agg }">
          <option v-for="agg in ['m', '5m', 'h', 'd']" :value="agg" :key="agg">
            {{ agg }}
          </option>
        </select>
      </form>
    </template>

    <template v-slot:kinds>
      <div v-for="(group, kind) in keynames" :key="kind" class="kindblock">
        <div class="kindname">
          <a :href="`#${kind}`">{{ kind }}</a>
        </div>
        <div class="keynames">
          <div v-for="(names, prefix) in group" :key="prefix" style="margin-top: 5px">
            <div v-for="keyname in names" :key="keyname" class="keyname">
              <a :href="`#${kind}:${keyname}`">{{ keyname }}</a>
            </div>
          </div>
        </div>
      </div>
    </template>

    <template v-slot:customs>
      <a href="#" @click.prevent="switchOrient"
        ><font-awesome-icon :icon="['far', 'window-maximize']" :rotation="orient ? '270' : null"
      /></a>
    </template>

    <template v-slot:content>
      <span v-if="nodata" class="nodata">No data</span>
      <div v-else>
        <div v-for="(group, kind) in charts" :key="kind">
          <a :name="`${kind}`"></a>
          <div v-for="(series, keyname) in group" :key="keyname">
            <p class="header">
              <a :name="`${kind}:${keyname}`"
                ><b>{{ kind }}:</b>{{ keyname }}</a
              >
            </p>
            <counts-chart :series="series" class="chart" :kind="kind" />
          </div>
        </div>
      </div>
    </template>
  </wrapper>
</template>

<script>
import _ from 'lodash'
import store from 'store2'
import { mapState } from 'vuex'
import ACTIONS from '@/store/action-types'
import MUTATIONS from '@/store/mutations-types.js'
import CountsChart from '@/components/CountsChart'
import Wrapper from '@/components/WrapperTable'

const ls = store.namespace('counts')

export default {
  components: {
    CountsChart,
    Wrapper
  },
  async created() {
    await Promise.all([this.$store.dispatch(ACTIONS.LOAD_ME), this.$store.dispatch(ACTIONS.LOAD_DASHBOARDS)])
    this.lognames = await this.$store.dispatch(ACTIONS.LOAD_COUNTS_LOGNAMES, this.dash.id)

    this.parseLocation()
    this.updateLocation()

    await this.updateCounts()
  },
  data() {
    return {
      lognames: [],
      stats: [],
      filters: {
        hostname: '',
        logname: '',
        pid: '',
        version: '',
        agg: ''
      },
      loading: true,
      paused: false,
      counts: null
    }
  },
  watch: {
    ['filters.logname'](v, prev) {
      document.title = this.dash.name + ' • ' + v
      if (prev !== '') {
        Object.assign(this.filters, {
          hostname: '',
          version: ''
        })
      }
      this.updateStats()
    }
  },
  computed: {
    ...mapState(['user', 'dashboards', 'orient']),
    dash() {
      return (this.dashboards || []).find(dash => dash.id === +this.$route.params.id)
    },
    sortedLognames() {
      return _.chain(this.lognames)
        .sortBy('cnt')
        .reverse()
        .map('logname')
        .value()
    },
    sortedHostnames() {
      return this.groupStatsBy('hostname')
    },
    sortedVersions() {
      return this.groupStatsBy('version', 'updated')
    },
    keynames() {
      return _.mapValues(this.charts, group => {
        return _.chain(group)
          .keys()
          .groupBy(v => v.split(':')[0])
          .value()
      })
    },
    charts() {
      if (!this.counts) {
        return null
      }
      return _.chain(this.counts)
        .groupBy('kind')
        .pick(['inc', 'avg', 'max', 'min', 'per', 'time'])
        .mapValues(group => {
          return _.chain(group)
            .sortBy('keyname')
            .groupBy('keyname')
            .mapValues(group => {
              return _.chain(group)
                .keyBy('hostname')
                .map(({ data }, hostname) => {
                  data = this.filled(data.map(([x, y]) => [x * 1000, y]).reverse())
                  const hex = this.convertToHex(hostname)
                  const color = '#' + hex.slice(0, 6)
                  return { name: hostname, data, color }
                })
                .sortBy('name')
                .value()
            })
            .value()
        })
        .value()
    },
    nodata() {
      return _.size(this.charts) === 0
    }
  },
  methods: {
    parseLocation() {
      let { hostname = '', logname = '', pid = '', version = '', agg = 'm' } = this.$route.query
      if (logname === '') {
        logname = ls.get(`dash${this.dash.id}.filters.logname`)
        logname = this.sortedLognames.includes(logname) ? logname : _.first(this.sortedLognames) || ''
      }
      this.filters = {
        logname,
        hostname,
        pid,
        version,
        agg
      }
    },
    convertToHex(str) {
      let hash = 0
      if (str.length === 0) {
        return ''
      }

      for (let i = 0; i < str.length; i++) {
        let char = str.charCodeAt(i)
        hash = (hash << 5) - hash + char
        hash = hash & hash // Convert to 32bit integer
      }

      return Math.abs(hash).toString(16)
    },
    filled(list) {
      if (list.length < 2) {
        return list
      }
      const filled = [] //list.slice(0, 1)
      const first = _.first(list)[0]
      const last = Date.now() //_.last(list)[0]
      const delta = {
        m: 1000 * 60,
        '5m': 1000 * 60 * 5,
        h: 1000 * 60 * 60,
        d: 1000 * 60 * 60 * 24
      }[this.filters.agg]
      let i = 0
      for (let t = first; t <= last; t += delta) {
        if (t === _.get(list, [i, 0])) {
          filled.push(list[i])
          i++
        } else {
          filled.push([t, null])
        }
      }
      return filled
    },
    async onChangeFilters(e) {
      console.log('onChangeFilters', e)
      ls.set(`dash${this.dash.id}.filters.logname`, this.filters.logname)

      await this.updateCounts()
      this.updateLocation()
    },
    updateLocation() {
      let query = { ...this.filters }
      query = _.pickBy(query)
      this.$router.replace({ query })
    },
    async updateStats() {
      if (this.stats.length) {
        this.stats = []
      }
      this.stats = await this.$store.dispatch(ACTIONS.LOAD_COUNTS_STATS, {
        dashId: this.dash.id,
        logname: this.filters.logname
      })
    },
    async updateCounts() {
      this.loading = true
      if (!this.filters.logname) {
        this.loading = false
        return
      }
      this.counts = await this.$store.dispatch(ACTIONS.LOAD_COUNTS, {
        ...this.filters,
        dash_id: this.dash.id
      })
      console.log('updateCounts', this.counts)
      this.loading = false
    },
    groupStatsBy(fieldname, sort = 'cnt') {
      return _.chain(this.stats)
        .groupBy(fieldname)
        .map((group, key) => {
          const cnt = _.sumBy(group, 'cnt')
          const updated = _.chain(group)
            .map('updated')
            .max()
            .value()
          return { [fieldname]: key, cnt, updated }
        })
        .sortBy(v => -v[sort])
        .map(fieldname)
        .value()
    },
    switchOrient() {
      this.$store.commit(MUTATIONS.SWITCH_ORIENT)
    }
  }
}
</script>

<style lang="scss" scoped>
input#filter-pid {
  display: inline-block;
  width: 40%;
}
select#filter-version {
  display: inline-block;
  width: 100px;
}
select#filter-agg {
  display: inline-block;
  width: 70px;
  float: right;
}
.chart {
  height: 240px;
  min-width: 360px;
  border-bottom: dashed 1px #000;
}
.header {
  font-size: 18px;
  margin-left: 8px;
  text-align: left;
}
.nodata {
  /*display: block;*/
  /*position: relative;*/
  /*top: 45%;*/
  /*text-align: center;*/
}
</style>
