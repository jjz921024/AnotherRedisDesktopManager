<template>
  <div>
    <!-- vxe table must get a container with a fixed height -->
    <div class="content-table-container">
      <vxe-table
        ref="contentTable"
        size="mini" max-height="100%" min-height="72px"
        border="default" stripe show-overflow="title"
        :scroll-y="{enabled: true}"
        :row-config="{isHover: true, height: 34}"
        :column-config="{resizable: true}"
        :empty-text="$t('el.table.emptyText')"
        :data="tairhashData">
        <vxe-column type="seq" :title="'ID (Total: ' + total + ')'" width="120"></vxe-column>
        <vxe-column field="field" title="Field" sortable>
          <template v-slot="scope">
            {{ $util.bufToString(scope.row.field) }}
          </template>
        </vxe-column>
        <vxe-column field="value" title="Value" sortable>
          <template v-slot="scope">
            {{ $util.cutString($util.bufToString(scope.row.value), 100) }}
          </template>
        </vxe-column>
        <vxe-column field="expire" :title="$t('message.expire')" width="120" sortable>
          <template v-slot="scope">
            {{ formatExpire(scope.row.expire) }}
          </template>
        </vxe-column>
        <vxe-column field="version" :title="$t('message.version')" width="100" sortable></vxe-column>
        <vxe-column title="Operate" width="166">
          <template slot-scope="scope" slot="header">
            <el-input size="mini"
              :placeholder="$t('message.key_to_search')"
              :suffix-icon="loadingIcon"
              @keyup.native.enter='initShow()'
              v-model="filterValue">
            </el-input>
          </template>
          <template slot-scope="scope">
            <el-button type="text" @click="$util.copyToClipboard(scope.row.value)" icon="el-icon-document" :title="$t('message.copy')"></el-button>
            <el-button type="text" @click="dumpCommand(scope.row)" icon="fa fa-code" :title="$t('message.dump_to_clipboard')"></el-button>
          </template>
        </vxe-column>
      </vxe-table>
    </div>

    <!-- load more content -->
    <div class='content-more-container'>
      <el-button
        size='mini'
        @click='initShow(false)'
        :icon='loadingIcon'
        :disabled='loadMoreDisable'
        class='content-more-btn'>
        {{ $t('message.load_more_keys') }}
      </el-button>
    </div>
  </div>
</template>

<script>
import { VxeTable, VxeColumn } from 'vxe-table';

export default {
  data() {
    return {
      total: 0,
      filterValue: '',
      tairhashData: [], // {field: xxx, value: xxx, expire: xxx, version: xxx}
      loadingIcon: '',
      pageSize: 200,
      searchPageSize: 2000,
      oneTimeListLength: 0,
      loadMoreDisable: false,
      allData: [], // store all data for filtering
    };
  },
  components: { VxeTable, VxeColumn },
  props: ['client', 'redisKey'],
  watch: {
    tairhashData(newValue, oldValue) {
      // scroll to bottom while loading more
      if (oldValue.length && (newValue.length > oldValue.length)) {
        setTimeout(() => {
          this.$refs.contentTable && this.$refs.contentTable.scrollTo(0, 99999999);
        }, 0);
      }
    },
  },
  methods: {
    initShow(resetTable = true) {
      if (resetTable) {
        this.resetTable();
        this.loadingIcon = 'el-icon-loading';

        // total lines
        this.initTotal();

        // load data using EXHGETALL
        this.loadTairHashData();
      } else {
        // load more data
        this.loadingIcon = 'el-icon-loading';
        this.oneTimeListLength = 0;
        this.loadMoreData();

        // Load expire and version for new data
        const start = this.tairhashData.length;
        const pageSize = this.filterValue ? this.searchPageSize : this.pageSize;
        const end = Math.min(start + pageSize, this.allData.length);
        const newData = this.allData.slice(start, end);

        if (newData.length > 0) {
          this.loadExpireAndVersion(
            newData.map(item => item.field),
            start
          );
        }
      }
    },
    initTotal() {
      this.client.call('EXHLEN', this.redisKey).then((reply) => {
        this.total = reply;
      }).catch((e) => {});
    },
    resetTable() {
      this.tairhashData = [];
      this.allData = [];
      this.oneTimeListLength = 0;
      this.loadMoreDisable = false;
    },
    formatExpire(ms) {
      // -1 means never expire (no expiration set)
      // -2 means key or field does not exist
      // Otherwise format milliseconds to human readable
      if (ms === undefined || ms === null) {
        return '-';
      }

      if (ms === -1) {
        return this.$t('message.never_expire');
      }

      if (ms === -2) {
        return this.$t('message.not_exist');
      }

      if (ms < 0) {
        return '-';
      }

      // Convert milliseconds to seconds
      const seconds = Math.floor(ms / 1000);

      if (seconds < 60) {
        return `${seconds}${this.$t('message.seconds')}`;
      } else if (seconds < 3600) {
        const minutes = Math.floor(seconds / 60);
        return `${minutes}${this.$t('message.minutes')}`;
      } else if (seconds < 86400) {
        const hours = Math.floor(seconds / 3600);
        return `${hours}${this.$t('message.hours')}`;
      } else {
        const days = Math.floor(seconds / 86400);
        return `${days}${this.$t('message.days')}`;
      }
    },
    async loadTairHashData() {
      try {
        // Use EXHGETALL to get all field-value pairs
        const reply = await this.client.call('EXHGETALL', this.redisKey);

        if (!reply || reply.length === 0) {
          this.loadingIcon = '';
          this.loadMoreDisable = true;
          return;
        }

        // EXHGETALL returns [field1, value1, field2, value2, ...]
        const rawData = [];
        for (let i = 0; i < reply.length; i += 2) {
          const field = reply[i];
          const value = reply[i + 1];

          // Filter by search keyword if provided
          if (this.filterValue) {
            const fieldStr = this.$util.bufToString(field).toLowerCase();
            const valueStr = this.$util.bufToString(value).toLowerCase();
            const keyword = this.filterValue.toLowerCase();

            if (!fieldStr.includes(keyword) && !valueStr.includes(keyword)) {
              continue;
            }
          }

          rawData.push({
            field: field,
            value: value,
            expire: null,
            version: null,
          });
        }

        this.allData = rawData;
        this.oneTimeListLength = 0;

        // Load first page
        this.loadMoreData();

        // Load expire and version info for first page
        if (this.tairhashData.length > 0) {
          this.loadExpireAndVersion(
            this.tairhashData.map(item => item.field),
            0
          );
        }
      } catch (e) {
        this.loadingIcon = '';
        this.loadMoreDisable = true;
        this.$message.error(e.message);
      }
    },
    loadMoreData() {
      // Check if all data is loaded
      if (this.tairhashData.length >= this.allData.length) {
        this.loadingIcon = '';
        this.loadMoreDisable = true;
        return;
      }

      const start = this.tairhashData.length;
      const pageSize = this.filterValue ? this.searchPageSize : this.pageSize;
      const end = Math.min(start + pageSize, this.allData.length);

      const newData = this.allData.slice(start, end);
      this.oneTimeListLength += newData.length;
      this.tairhashData = this.tairhashData.concat(newData);

      // Check if more data available
      if (this.tairhashData.length >= this.allData.length) {
        this.loadingIcon = '';
        this.loadMoreDisable = true;
      } else {
        this.loadingIcon = '';
      }
    },
    async loadExpireAndVersion(fields, startIndex) {
      if (!fields || fields.length === 0) {
        return;
      }

      try {
        // Use Promise.all to fetch EXHPTTL and EXHVER in parallel for all fields
        const promises = fields.map((field) => {
          const fieldStr = this.$util.bufToString(field);
          return Promise.all([
            this.client.call('EXHPTTL', this.redisKey, fieldStr).catch(() => -1),
            this.client.call('EXHVER', this.redisKey, fieldStr).catch(() => null),
          ]);
        });

        const results = await Promise.all(promises);

        // Update expire and version in tairhashData
        // Note: Redis returns values as strings, need to parse to integers
        results.forEach((result, index) => {
          const dataIndex = startIndex + index;
          if (dataIndex < this.tairhashData.length) {
            this.$set(this.tairhashData, dataIndex, {
              ...this.tairhashData[dataIndex],
              expire: parseInt(result[0]),
              version: parseInt(result[1]),
            });
          }
        });
      } catch (e) {
        // Silently handle errors for individual field queries
        console.error('Error loading expire/version:', e);
      }
    },
    dumpCommand(item) {
      const lines = item ? [item] : this.tairhashData;
      const params = lines.map(line => {
        const fieldStr = this.$util.bufToQuotation(line.field);
        const valueStr = this.$util.bufToQuotation(line.value);
        const expirePart = line.expire && line.expire > 0 ? ` EX ${Math.floor(line.expire / 1000)}` : '';
        const versionPart = line.version && line.version > 0 ? ` VER ${line.version}` : '';
        return `${fieldStr} ${valueStr}${expirePart}${versionPart}`;
      });

      const command = `EXHSET ${this.$util.bufToQuotation(this.redisKey)} ${params.join(' ')}`;
      this.$util.copyToClipboard(command);
      this.$message.success({ message: this.$t('message.copy_success'), duration: 800 });
    },
  },
  mounted() {
    this.initShow();
  },
};
</script>
