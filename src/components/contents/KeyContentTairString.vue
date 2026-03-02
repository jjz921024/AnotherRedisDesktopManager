<template>
  <div class="key-content-tairstring">
    <!-- metadata display -->
    <div class="tairstring-metadata">
      <el-row :gutter="20">
        <el-col :span="12">
          <div class="metadata-item">
            <span class="metadata-label">{{ $t('message.version') }}:</span>
            <span class="metadata-value">{{ version || '-' }}</span>
          </div>
        </el-col>
        <el-col :span="12">
          <div class="metadata-item">
            <span class="metadata-label">{{ $t('message.expire') }}:</span>
            <span class="metadata-value">{{ formatExpire(expire) }}</span>
          </div>
        </el-col>
      </el-row>
    </div>

    <!-- value display -->
    <el-form>
      <el-form-item>
        <FormatViewer
          ref="formatViewer"
          :content="content"
          :binary="binary"
          :redisKey="redisKey"
          float="">
        </FormatViewer>
      </el-form-item>

      <!-- operate buttons -->
      <div class="tairstring-operations">
        <el-button
          type="primary"
          size="small"
          @click="$util.copyToClipboard(content)"
          icon="el-icon-document">
          {{ $t('message.copy') }}
        </el-button>
        <el-button
          type="primary"
          size="small"
          @click="dumpCommand"
          icon="fa fa-code">
          {{ $t('message.dump_to_clipboard') }}
        </el-button>
      </div>
    </el-form>
  </div>
</template>

<script>
import FormatViewer from '@/components/FormatViewer';

export default {
  data() {
    return {
      content: Buffer.from(''),
      binary: false,
      version: null,
      expire: null,
    };
  },
  props: ['client', 'redisKey', 'hotKeyScope'],
  components: { FormatViewer },
  methods: {
    initShow() {
      // Fetch value and version using EXGET
      this.client.call('EXGET', this.redisKey).then((reply) => {
        if (!reply) {
          this.$message.error(this.$t('message.key_not_exists'));
          return;
        }

        // EXGET returns [value, version]
        this.content = reply[0];
        this.version = parseInt(reply[1]);
      }).catch((e) => {
        this.$message.error(e.message);
      });

      // Fetch TTL using PTTL
      this.client.call('PTTL', this.redisKey).then((reply) => {
        this.expire = parseInt(reply);
      }).catch((e) => {
        // Silently handle PTTL errors
        console.error('PTTL error:', e);
      });
    },
    formatExpire(ms) {
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
    dumpCommand() {
      const valueStr = this.$util.bufToQuotation(this.content);
      const expirePart = this.expire && this.expire > 0 ? ` EX ${Math.floor(this.expire / 1000)}` : '';
      const versionPart = this.version && this.version > 0 ? ` ABS ${this.version}` : '';

      const command = `EXSET ${this.$util.bufToQuotation(this.redisKey)} ${valueStr}${expirePart}${versionPart}`;
      this.$util.copyToClipboard(command);
      this.$message.success({ message: this.$t('message.copy_success'), duration: 800 });
    },
  },
  mounted() {
    this.initShow();
  },
};
</script>

<style scoped>
.key-content-tairstring {
  padding: 0;
}

.tairstring-metadata {
  background: var(--bg-color, #f5f7fa);
  border-radius: 4px;
  padding: 12px 16px;
  margin-bottom: 12px;
}

.metadata-item {
  display: flex;
  align-items: center;
}

.metadata-label {
  font-weight: 600;
  color: var(--text-color, #606266);
  margin-right: 8px;
}

.metadata-value {
  color: var(--text-color-regular, #909399);
}

.tairstring-operations {
  margin-top: 12px;
  display: flex;
  gap: 8px;
}
</style>
