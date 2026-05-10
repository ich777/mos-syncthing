<template>
  <div>
    <h2 class="mb-4">{{ $t('plugin_syncthing.title') }}</h2>

    <v-skeleton-loader v-if="loading" :loading="true" type="card" />

    <div v-else style="margin-bottom: 80px">
      <!-- Status Card -->
      <v-card class="mb-4 pa-0">
        <v-card-title class="d-flex align-center">
          <v-icon class="mr-2">mdi-sync</v-icon>
          <span>{{ $t('plugin_syncthing.status') }}</span>
        </v-card-title>
        <v-card-text>
          <v-row>
            <v-col cols="12" md="3">
              <span class="text-subtitle-1 font-weight-medium">
                {{ $t('plugin_syncthing.version') }} {{ currentVersion || $t('plugin_syncthing.not_installed') }}
              </span>
            </v-col>
            <v-col cols="12" md="3">
              <span class="text-subtitle-1 font-weight-medium">
                {{ $t('plugin_syncthing.latest_version') }} {{ latestVersion || '-' }}
              </span>
            </v-col>
            <v-col v-if="webuiEnabled" cols="12" md="3">
              <span class="text-subtitle-1 font-weight-medium">
                {{ $t('plugin_syncthing.status') }}: {{ running ? $t('plugin_syncthing.running') : $t('plugin_syncthing.not_running') }}
              </span>
            </v-col>
            <v-col cols="12" md="auto" class="d-flex ga-2 flex-nowrap flex-md-row">
              <v-btn size="small" variant="tonal" color="secondary" @click="checkForUpdates" :loading="checkingUpdates">
                {{ $t('plugin_syncthing.check_updates') }}
              </v-btn>
              <v-btn
                v-if="!currentVersion"
                size="small" variant="tonal" color="primary"
                @click="doInstall" :loading="installing"
              >
                {{ $t('plugin_syncthing.install') }}
              </v-btn>
              <v-btn
                v-else-if="currentVersion && updateAvailable"
                size="small" variant="tonal" color="primary"
                @click="doUpdate" :loading="updating"
              >
                {{ $t('plugin_syncthing.update') }}
              </v-btn>
            </v-col>
          </v-row>
        </v-card-text>
      </v-card>

      <!-- Settings Card -->
      <v-card class="mb-4 pa-0">
        <v-card-title class="d-flex align-center">
          <v-icon class="mr-2">mdi-cog</v-icon>
          <span>{{ $t('plugin_syncthing.settings') }}</span>
        </v-card-title>
        <v-card-text>
          <v-switch
            v-if="webuiEnabled"
            v-model="settings.auto_start"
            :label="$t('plugin_syncthing.auto_start')"
            inset color="green" hide-details
          />
          <v-text-field
            v-model="settings.webui_ip"
            :label="$t('plugin_syncthing.webui_ip')"
            :hint="$t('plugin_syncthing.default_ip_hint')"
            persistent-hint
            class="mt-4"
            style="max-width: 200px"
          />
          <v-text-field
            v-model="settings.webui_port"
            :label="$t('plugin_syncthing.webui_port')"
            :hint="!webuiEnabled ? $t('plugin_syncthing.default_port_hint', { port: 8384 }) : ''"
            :persistent-hint="!webuiEnabled"
            class="mt-4"
            type="number"
            style="max-width: 200px"
          />
          <v-btn color="primary" class="mt-4" @click="saveSettings" :loading="saving">
            <v-icon start>mdi-content-save</v-icon>
            {{ $t('plugin_syncthing.save_settings') }}
          </v-btn>
        </v-card-text>
      </v-card>

      <!-- Start/Stop Card -->
      <v-card v-if="currentVersion && webuiEnabled" class="mb-4 pa-0">
        <v-card-text class="d-flex align-center ga-2">
          <v-btn color="primary" rounded :loading="starting" @click="startDaemon">
            <v-icon start>mdi-play</v-icon>
            {{ $t('plugin_syncthing.start') }}
          </v-btn>
          <v-btn color="error" rounded variant="outlined" :loading="stopping" @click="stopDaemon">
            <v-icon start>mdi-stop</v-icon>
            {{ $t('plugin_syncthing.stop') }}
          </v-btn>
          <v-btn
            v-if="running"
            color="secondary" rounded variant="tonal"
            :href="webuiUrl" target="_blank"
          >
            <v-icon start>mdi-open-in-new</v-icon>
            {{ $t('plugin_syncthing.open_webui') }}
          </v-btn>
        </v-card-text>
      </v-card>
    </div>

    <!-- Overlay -->
    <v-overlay :model-value="overlay" class="align-center justify-center">
      <v-progress-circular color="onPrimary" size="64" indeterminate />
    </v-overlay>
  </div>
</template>

<script setup>
import { ref, reactive, computed, onMounted, onUnmounted } from 'vue';

const PLUGIN_NAME = 'syncthing';

const loading = ref(true);
const saving = ref(false);
const overlay = ref(false);
const starting = ref(false);
const stopping = ref(false);
const checkingUpdates = ref(false);
const updating = ref(false);
const installing = ref(false);
const statusInterval = ref(null);

const running = ref(false);
const currentVersion = ref('');
const latestVersion = ref('');
const updateAvailable = ref(false);

const settings = reactive({
  auto_start: false,
  webui_ip: '0.0.0.0',
  webui_port: '8384',
  version: ''
});

const webuiEnabled = computed(() => {
  const port = parseInt(settings.webui_port, 10);
  return port > 0;
});

const webuiUrl = computed(() => {
  const port = settings.webui_port || '8384';
  return `${window.location.protocol}//${window.location.hostname}:${port}`;
});

const getAuthHeaders = () => ({
  Authorization: 'Bearer ' + localStorage.getItem('authToken'),
});

const doInstall = async () => {
  installing.value = true;
  try {
    const res = await fetch('/api/v1/mos/plugins/query', {
      method: 'POST',
      headers: {
        ...getAuthHeaders(),
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        command: 'syncthing',
        args: ['install_binary'],
        timeout: 120,
        parse_json: false,
      }),
    });

    if (!res.ok) {
      throw new Error('Install failed');
    }

    await fetchSettings();
    await checkStatus();
    await checkForUpdates();
  } catch (e) {
    console.error('Failed to install:', e);
    alert('Failed to install Syncthing. Check logs for details.');
  } finally {
    installing.value = false;
  }
};

const doUpdate = async () => {
  updating.value = true;
  try {
    if (running.value) {
      await fetch('/api/v1/mos/plugins/query', {
        method: 'POST',
        headers: {
          ...getAuthHeaders(),
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({
          command: 'syncthing',
          args: ['stop'],
          timeout: 30,
          parse_json: false,
        }),
      });
    }

    const res = await fetch('/api/v1/mos/plugins/query', {
      method: 'POST',
      headers: {
        ...getAuthHeaders(),
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        command: 'syncthing',
        args: ['install_binary'],
        timeout: 120,
        parse_json: false,
      }),
    });

    if (!res.ok) {
      throw new Error('Update failed');
    }

    if (running.value) {
      await fetch('/api/v1/mos/plugins/query', {
        method: 'POST',
        headers: {
          ...getAuthHeaders(),
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({
          command: 'syncthing',
          args: ['start'],
          timeout: 30,
          parse_json: false,
        }),
      });
    }

    await fetchSettings();
    await checkStatus();
    await checkForUpdates();
  } catch (e) {
    console.error('Failed to update:', e);
    alert('Failed to update Syncthing. Check logs for details.');
  } finally {
    updating.value = false;
  }
};

const fetchSettings = async () => {
  try {
    const res = await fetch(`/api/v1/mos/plugins/settings/${PLUGIN_NAME}`, {
      headers: getAuthHeaders(),
    });
    if (res.ok) {
      const data = await res.json();
      if (data.auto_start !== undefined) {
        settings.auto_start = data.auto_start;
      }
      if (data.webui_ip !== undefined) {
        settings.webui_ip = data.webui_ip;
      }
      if (data.webui_port !== undefined) {
        settings.webui_port = data.webui_port;
      }
      if (data.version) {
        currentVersion.value = data.version;
      }
    }
  } catch (e) {
    console.error('Failed to fetch settings:', e);
  }
};

const checkStatus = async () => {
  try {
    const res = await fetch('/api/v1/mos/plugins/query', {
      method: 'POST',
      headers: {
        ...getAuthHeaders(),
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        command: 'syncthing',
        args: ['status'],
        timeout: 5,
        parse_json: true,
      }),
    });
    if (res.ok) {
      const data = await res.json();
      if (data.success && data.output) {
        running.value = data.output.running === true;
      }
    }
  } catch (e) {
    running.value = false;
  }
};

const checkForUpdates = async () => {
  checkingUpdates.value = true;
  try {
    const res = await fetch('/api/v1/mos/plugins/query', {
      method: 'POST',
      headers: {
        ...getAuthHeaders(),
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        command: 'syncthing',
        args: ['check_version'],
        timeout: 15,
        parse_json: true,
      }),
    });
    if (res.ok) {
      const data = await res.json();
      if (data.success && data.output) {
        latestVersion.value = data.output.latest || '';
        updateAvailable.value = currentVersion.value && data.output.update_available === true;
      }
    }
  } catch (e) {
    console.error('Failed to check updates:', e);
  } finally {
    checkingUpdates.value = false;
  }
};

const startDaemon = async () => {
  starting.value = true;
  try {
    await fetch('/api/v1/mos/plugins/query', {
      method: 'POST',
      headers: {
        ...getAuthHeaders(),
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        command: 'syncthing',
        args: ['start'],
        timeout: 30,
        parse_json: false,
      }),
    });
    await checkStatus();
  } catch (e) {
    console.error('Failed to start:', e);
  } finally {
    starting.value = false;
  }
};

const stopDaemon = async () => {
  stopping.value = true;
  try {
    await fetch('/api/v1/mos/plugins/query', {
      method: 'POST',
      headers: {
        ...getAuthHeaders(),
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        command: 'syncthing',
        args: ['stop'],
        timeout: 30,
        parse_json: false,
      }),
    });
    await checkStatus();
  } catch (e) {
    console.error('Failed to stop:', e);
  } finally {
    stopping.value = false;
  }
};

const saveSettings = async () => {
  saving.value = true;
  try {
    await fetch(`/api/v1/mos/plugins/settings/${PLUGIN_NAME}`, {
      method: 'POST',
      headers: {
        ...getAuthHeaders(),
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        auto_start: settings.auto_start,
        webui_ip: settings.webui_ip,
        webui_port: settings.webui_port,
        version: currentVersion.value,
      }),
    });
  } catch (e) {
    console.error('Failed to save settings:', e);
  } finally {
    saving.value = false;
  }
};

onMounted(async () => {
  try {
    await fetchSettings();
    await checkStatus();
    if (!currentVersion.value) {
      checkForUpdates();
    }

    if (webuiEnabled.value) {
      statusInterval.value = setInterval(async () => {
        await checkStatus();
      }, 5000);
    }
  } catch (e) {
    console.error('Failed to initialize:', e);
  } finally {
    loading.value = false;
  }
});

onUnmounted(() => {
  if (statusInterval.value) {
    clearInterval(statusInterval.value);
  }
});
</script>
