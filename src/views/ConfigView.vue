<script setup lang="ts">
import { ref, computed, watch, onMounted, onUnmounted, onActivated } from "vue";
import { useRoute } from "vue-router";
import SLSpinner from "../components/common/SLSpinner.vue";
import SLSwitch from "../components/common/SLSwitch.vue";
import SLSelect from "../components/common/SLSelect.vue";
import SLButton from "../components/common/SLButton.vue";
import { configApi } from "../api/config";
import { serverApi } from "../api/server";
import { m_pluginApi, type m_PluginInfo, type m_PluginConfigFile } from "../api/mcs_plugins";
import type { ConfigEntry as ConfigEntryType } from "../api/config";
import { useServerStore } from "../stores/serverStore";
import { i18n } from "../language";
import {
  Power,
  Trash2,
  RefreshCw,
  Settings,
  FileText,
  RotateCcw,
  FolderOpen,
  Edit,
} from "lucide-vue-next";

import ConfigCategories from "../components/config/ConfigCategories.vue";
import { systemApi } from "../api/system";
import "../styles/plugin-list.css";

const route = useRoute();
const store = useServerStore();

const entries = ref<ConfigEntryType[]>([]);
const editValues = ref<Record<string, string>>({});
const loading = ref(false);
const saving = ref(false);
const error = ref<string | null>(null);
const successMsg = ref<string | null>(null);
const searchQuery = ref("");
const activeCategory = ref("all");
const serverPath = computed(() => {
  const server = store.servers.find((s) => s.id === store.currentServerId);
  return server?.path || "";
});

const plugins = ref<m_PluginInfo[]>([]);
const pluginsLoading = ref(false);
const selectedPlugin = ref<m_PluginInfo | null>(null);
const activeTab = ref<"properties" | "plugins">("properties");
const loadingDebounceTimer = ref<number | null>(null);

const autoSaveDebounceTimer = ref<number | null>(null);
const AUTO_SAVE_DELAY = 1000;
const MEMORY_KEYS = new Set(["max_memory", "min_memory"]);
const pendingChangedKeys = ref<Set<string>>(new Set());

const currentServerId = computed(() => store.currentServerId);
const currentServer = computed(() => store.servers.find((s) => s.id === store.currentServerId));

const memoryEntries = computed<ConfigEntryType[]>(() => [
  {
    key: "max_memory",
    value: editValues.value.max_memory ?? "",
    description: i18n.t("config.memory_max_desc"),
    value_type: "number",
    default_value: currentServer.value ? String(currentServer.value.max_memory) : "",
    category: "performance",
  },
  {
    key: "min_memory",
    value: editValues.value.min_memory ?? "",
    description: i18n.t("config.memory_min_desc"),
    value_type: "number",
    default_value: currentServer.value ? String(currentServer.value.min_memory) : "",
    category: "performance",
  },
]);

const entriesWithMemory = computed(() => {
  const nonMemoryEntries = entries.value.filter((entry) => !MEMORY_KEYS.has(entry.key));
  return [...nonMemoryEntries, ...memoryEntries.value];
});

const categories = computed(() => {
  const cats = new Set(entriesWithMemory.value.map((e) => e.category));
  return ["all", ...Array.from(cats)];
});

const gamemodeOptions = ref([
  { label: i18n.t("config.gamemode.survival"), value: "survival" },
  { label: i18n.t("config.gamemode.creative"), value: "creative" },
  { label: i18n.t("config.gamemode.adventure"), value: "adventure" },
  { label: i18n.t("config.gamemode.spectator"), value: "spectator" },
]);

const difficultyOptions = ref([
  { label: i18n.t("config.difficulty.peaceful"), value: "peaceful" },
  { label: i18n.t("config.difficulty.easy"), value: "easy" },
  { label: i18n.t("config.difficulty.normal"), value: "normal" },
  { label: i18n.t("config.difficulty.hard"), value: "hard" },
]);

const filteredEntries = computed(() => {
  return entriesWithMemory.value.filter((e: ConfigEntryType) => {
    const matchCat = activeCategory.value === "all" || e.category === activeCategory.value;
    const matchSearch =
      !searchQuery.value ||
      e.key.toLowerCase().includes(searchQuery.value.toLowerCase()) ||
      (e.description ?? "").toLowerCase().includes(searchQuery.value.toLowerCase());
    return matchCat && matchSearch;
  });
});

onMounted(async () => {
  await store.refreshList();
  const routeId = route.params.id as string;
  if (routeId) {
    store.setCurrentServer(routeId);
  } else if (!store.currentServerId && store.servers.length > 0) {
    store.setCurrentServer(store.servers[0].id);
  }
  await loadProperties();
});

onUnmounted(() => {
  if (autoSaveDebounceTimer.value) {
    clearTimeout(autoSaveDebounceTimer.value);
  }
  pendingChangedKeys.value.clear();
});

watch(
  () => store.currentServerId,
  async () => {
    if (store.currentServerId) {
      await loadProperties();
    }
  },
);

function syncMemoryEditValues(target: Record<string, string>) {
  if (!currentServer.value) {
    target.max_memory = "";
    target.min_memory = "";
    return;
  }

  target.max_memory = String(currentServer.value.max_memory);
  target.min_memory = String(currentServer.value.min_memory);
}

function parsePositiveMemory(value: string): number | null {
  const memory = Number.parseInt(value, 10);
  if (!Number.isFinite(memory) || memory <= 0) {
    return null;
  }
  return memory;
}

function getEntryDescription(entry: ConfigEntryType): string {
  if (entry.key === "max_memory") {
    return i18n.t("config.memory_max_desc");
  }
  if (entry.key === "min_memory") {
    return i18n.t("config.memory_min_desc");
  }
  const key = `config.properties.${entry.key}`;
  const description = i18n.t(key);
  return description === key ? "" : description;
}

async function loadProperties() {
  if (!serverPath.value) return;

  if (loadingDebounceTimer.value) {
    clearTimeout(loadingDebounceTimer.value);
  }

  loading.value = true;
  error.value = null;
  try {
    const result = await configApi.readServerProperties(serverPath.value);
    entries.value = result.entries as ConfigEntryType[];
    const nextValues = { ...result.raw };
    syncMemoryEditValues(nextValues);
    editValues.value = nextValues;
  } catch (e) {
    error.value = String(e);
    entries.value = [];
    const nextValues: Record<string, string> = {};
    syncMemoryEditValues(nextValues);
    editValues.value = nextValues;
  } finally {
    pendingChangedKeys.value.clear();
    loading.value = false;
  }
}

function updateValue(key: string, value: string | number | boolean) {
  editValues.value[key] = String(value);
  pendingChangedKeys.value.add(key);

  // 启动自动保存防抖
  if (autoSaveDebounceTimer.value) {
    clearTimeout(autoSaveDebounceTimer.value);
  }

  autoSaveDebounceTimer.value = window.setTimeout(() => {
    void autoSaveProperties();
  }, AUTO_SAVE_DELAY);
}

async function autoSaveProperties() {
  if (!currentServerId.value) return;

  const changedKeys = Array.from(pendingChangedKeys.value);
  pendingChangedKeys.value.clear();
  if (changedKeys.length === 0) return;

  const shouldWriteProperties =
    !!serverPath.value && entries.value.length > 0 && changedKeys.some((key) => !MEMORY_KEYS.has(key));
  const shouldValidateMemory = changedKeys.some((key) => MEMORY_KEYS.has(key));

  const maxMemory = parsePositiveMemory(editValues.value.max_memory ?? "");
  const minMemory = parsePositiveMemory(editValues.value.min_memory ?? "");
  const shouldUpdateMemory =
    shouldValidateMemory &&
    maxMemory !== null &&
    minMemory !== null &&
    minMemory <= maxMemory &&
    !!currentServer.value &&
    (currentServer.value.max_memory !== maxMemory || currentServer.value.min_memory !== minMemory);

  if (!shouldWriteProperties && !shouldUpdateMemory) return;

  saving.value = true;
  error.value = null;
  successMsg.value = null;

  let firstError: string | null = null;
  let hasSuccess = false;

  if (shouldWriteProperties && serverPath.value) {
    const propertyValues = Object.fromEntries(
      Object.entries(editValues.value).filter(([key]) => !MEMORY_KEYS.has(key)),
    );
    try {
      await configApi.writeServerProperties(serverPath.value, propertyValues);
      hasSuccess = true;
    } catch (e) {
      firstError = String(e);
    }
  }

  if (shouldUpdateMemory && maxMemory !== null && minMemory !== null) {
    try {
      await serverApi.updateServerMemory(currentServerId.value, maxMemory, minMemory);
      await store.refreshList();
      hasSuccess = true;
    } catch (e) {
      if (!firstError) {
        firstError = String(e);
      }
    }
  }

  if (firstError) {
    error.value = firstError;
  } else if (hasSuccess) {
    successMsg.value = i18n.t("config.saved");
    setTimeout(() => (successMsg.value = null), 3000);
  }

  saving.value = false;
}

function handleCategoryChange(category: string) {
  activeCategory.value = category;
  window.scrollTo({ top: 0, behavior: "smooth" });
}

function handleSearchUpdate(value: string) {
  searchQuery.value = value;
}

async function loadPlugins() {
  if (!store.currentServerId) return;

  if (loadingDebounceTimer.value) {
    clearTimeout(loadingDebounceTimer.value);
  }

  pluginsLoading.value = true;
  error.value = null;
  try {
    plugins.value = await m_pluginApi.m_getPlugins(store.currentServerId);
  } catch (e) {
    error.value = String(e);
    plugins.value = [];
  } finally {
    pluginsLoading.value = false;
  }
}

async function togglePlugin(plugin: m_PluginInfo) {
  if (!store.currentServerId) return;

  if (!plugin.file_name.endsWith(".jar") && !plugin.file_name.endsWith(".jar.disabled")) {
    alert(i18n.t("config.not_jar_file", { file: plugin.file_name }));
    return;
  }

  try {
    await m_pluginApi.m_togglePlugin(store.currentServerId, plugin.file_name, !plugin.enabled);
    plugin.enabled = !plugin.enabled;
  } catch (e) {
    error.value = String(e);
  }
}

async function deletePlugin(plugin: m_PluginInfo) {
  if (!store.currentServerId) return;
  if (!confirm(i18n.t("config.confirm_delete_plugin", { name: plugin.name }))) return;
  try {
    await m_pluginApi.m_deletePlugin(store.currentServerId, plugin.file_name);
    plugins.value = plugins.value.filter((p) => p.file_name !== plugin.file_name);
    if (selectedPlugin.value?.file_name === plugin.file_name) {
      selectedPlugin.value = null;
    }
  } catch (e) {
    error.value = String(e);
  }
}

async function reloadPlugins() {
  if (!store.currentServerId) return;
  try {
    await m_pluginApi.m_reloadPlugins(store.currentServerId);
    await loadPlugins();
  } catch (e) {
    error.value = String(e);
  }
}

function handlePluginClick(plugin: m_PluginInfo) {
  if (selectedPlugin.value?.file_name === plugin.file_name) {
    selectedPlugin.value = null;
  } else {
    selectedPlugin.value = plugin;
  }
}

async function openPluginFolder(plugin: m_PluginInfo) {
  if (!store.currentServerId) return;
  const server = store.servers.find((s) => s.id === store.currentServerId);
  if (!server) return;

  const basePath = server.path.replace(/[/\\]$/, "");
  const pluginConfigPath = `${basePath}${basePath.includes("\\") ? "\\" : "/"}plugins${basePath.includes("\\") ? "\\" : "/"}${plugin.m_id}`;

  try {
    await systemApi.openFolder(pluginConfigPath);
  } catch (e) {
    error.value = String(e);
  }
}

async function openConfigFile(config: m_PluginConfigFile) {
  try {
    await systemApi.openFile(config.file_path);
  } catch (e) {
    error.value = String(e);
  }
}

function formatFileSize(bytes: number) {
  if (bytes < 1024) return bytes + " B";
  if (bytes < 1024 * 1024) return (bytes / 1024).toFixed(2) + " KB";
  return (bytes / (1024 * 1024)).toFixed(2) + " MB";
}

watch(
  () => store.currentServerId,
  async () => {
    if (store.currentServerId) {
      await loadProperties();
      await loadPlugins();
    }
  },
);

onActivated(async () => {
  await loadProperties();
  await loadPlugins();
});
</script>

<template>
  <div class="config-view animate-fade-in-up">
    <div class="config-header">
      <div class="server-path-display text-mono text-caption">
        {{ serverPath }}/server.properties
      </div>
      <div class="tab-switcher">
        <button
          type="button"
          class="tab-button"
          :class="{ active: activeTab === 'properties' }"
          @click="activeTab = 'properties'"
        >
          {{ i18n.t("config.server_properties") }}
        </button>
        <button
          type="button"
          class="tab-button"
          :class="{ active: activeTab === 'plugins' }"
          @click="activeTab = 'plugins'"
        >
          {{ i18n.t("config.server_plugins") }}
        </button>
      </div>
    </div>

    <div v-if="!currentServerId" class="empty-state">
      <p class="text-body">{{ i18n.t("config.no_server") }}</p>
    </div>

    <template v-else>
      <div v-if="error" class="error-banner">
        <span>{{ error }}</span>
        <button class="banner-close" @click="error = null">x</button>
      </div>
      <div v-if="successMsg" class="success-banner">
        <span>{{ i18n.t("config.saved") }}</span>
      </div>

      <template v-if="activeTab === 'properties'">
        <ConfigCategories
          :categories="categories"
          :activeCategory="activeCategory"
          :searchQuery="searchQuery"
          @updateCategory="handleCategoryChange"
          @updateSearch="handleSearchUpdate"
        />

        <div v-if="loading" class="loading-state">
          <SLSpinner size="lg" />
          <span>{{ i18n.t("config.loading") }}</span>
        </div>

        <div v-else class="config-entries">
          <div v-for="entry in filteredEntries" :key="entry.key" class="config-entry glass-card">
            <div class="entry-header">
              <div class="entry-key-row">
                <span class="entry-key text-mono">{{ entry.key }}</span>
              </div>
              <p v-if="getEntryDescription(entry)" class="entry-desc text-caption">
                {{ getEntryDescription(entry) }}
              </p>
            </div>
            <div class="entry-control">
              <template
                v-if="
                  entry.value_type === 'boolean' ||
                  editValues[entry.key] === 'true' ||
                  editValues[entry.key] === 'false'
                "
              >
                <SLSwitch
                  :modelValue="editValues[entry.key] === 'true'"
                  @update:modelValue="updateValue(entry.key, $event)"
                />
              </template>
              <template v-else-if="entry.key === 'gamemode'">
                <SLSelect
                  :modelValue="editValues[entry.key]"
                  :options="gamemodeOptions"
                  @update:modelValue="updateValue(entry.key, $event)"
                  style="width: 200px"
                />
              </template>
              <template v-else-if="entry.key === 'difficulty'">
                <SLSelect
                  :modelValue="editValues[entry.key]"
                  :options="difficultyOptions"
                  @update:modelValue="updateValue(entry.key, $event)"
                  style="width: 200px"
                />
              </template>
              <template v-else>
                <input
                  :value="editValues[entry.key]"
                  type="text"
                  :placeholder="entry.default_value"
                  @input="
                    (e) => {
                      const value = (e.target as HTMLInputElement).value;
                      if (value === '' || /^\d+$/.test(value)) {
                        updateValue(entry.key, value);
                      }
                    }
                  "
                  class="input integer-input"
                />
              </template>
            </div>
          </div>
          <div v-if="filteredEntries.length === 0 && !loading" class="empty-state">
            <p class="text-caption">{{ i18n.t("config.no_config") }}</p>
          </div>
        </div>
      </template>

      <template v-if="activeTab === 'plugins'">
        <div class="plugins-header">
          <h3>{{ i18n.t("config.server_plugins") }}</h3>
          <div class="plugins-header-actions">
            <SLButton @click="loadPlugins" :loading="pluginsLoading" variant="secondary" size="sm">
              <RotateCcw :size="16" />
              {{ i18n.t("config.refresh_list") }}
            </SLButton>
            <SLButton
              @click="reloadPlugins"
              :loading="pluginsLoading"
              variant="danger"
              size="sm"
              class="reload-btn"
              title="重载插件可能导致服务器异常，部分插件可能不支持"
            >
              <RefreshCw :size="14" />
              {{ i18n.t("config.reload_plugins") }}
            </SLButton>
          </div>
        </div>

        <div v-if="pluginsLoading" class="loading-state">
          <SLSpinner size="lg" />
          <span>{{ i18n.t("config.loading_plugins") }}</span>
        </div>

        <div v-else class="plugins-container">
          <div v-if="plugins.length === 0" class="empty-state">
            <p class="text-caption">{{ i18n.t("config.no_plugins") }}</p>
          </div>

          <div v-else class="plugin-list-view">
            <div
              v-for="plugin in plugins"
              :key="plugin.file_name"
              class="plugin-list-item"
              :class="{
                disabled: !plugin.enabled,
                expanded: selectedPlugin?.file_name === plugin.file_name,
              }"
              @click="handlePluginClick(plugin)"
            >
              <div class="plugin-list-icon">
                {{ plugin.name.charAt(0).toUpperCase() }}
              </div>
              <div class="plugin-list-info">
                <div class="plugin-list-header">
                  <h4>{{ plugin.name }}</h4>
                  <span class="plugin-list-version">{{ plugin.version }}</span>
                  <div v-if="plugin.has_config_folder" class="config-badge">
                    <Settings :size="14" />
                    {{ plugin.config_files.length }} {{ i18n.t("config.config_files_count") }}
                  </div>
                  <div v-else class="no-config-badge">
                    <FileText :size="14" />
                    {{ i18n.t("config.no_config_files") }}
                  </div>
                </div>
                <div
                  v-if="selectedPlugin?.file_name === plugin.file_name"
                  class="plugin-list-details"
                >
                  <p>{{ i18n.t("config.author") }}: {{ plugin.author }}</p>
                  <p v-if="plugin.description">{{ plugin.description }}</p>
                  <p>{{ formatFileSize(plugin.file_size) }}</p>

                  <div v-if="plugin.has_config_folder" class="plugin-config-section">
                    <div class="plugin-config-section-header">
                      <h5>{{ i18n.t("config.config_files") }}</h5>
                      <SLButton
                        size="sm"
                        variant="secondary"
                        @click.stop="openPluginFolder(plugin)"
                      >
                        <FolderOpen :size="14" />
                        {{ i18n.t("common.open_folder") }}
                      </SLButton>
                    </div>
                    <div v-if="plugin.config_files.length > 0" class="plugin-config-files-list">
                      <div
                        v-for="config in plugin.config_files"
                        :key="config.file_name"
                        class="plugin-config-file-item"
                        @click.stop="openConfigFile(config)"
                      >
                        <div class="plugin-config-file-name">{{ config.file_name }}</div>
                        <div class="plugin-config-file-type">{{ config.file_type }}</div>
                        <div class="plugin-config-file-actions">
                          <SLButton size="sm" variant="secondary">
                            <Edit :size="14" />
                            {{ i18n.t("config.open") }}
                          </SLButton>
                        </div>
                      </div>
                    </div>
                    <div v-else class="empty-state">
                      <p class="text-caption">{{ i18n.t("config.empty_config_folder") }}</p>
                    </div>
                  </div>
                </div>
              </div>
              <div class="plugin-list-actions">
                <SLButton
                  @click.stop="togglePlugin(plugin)"
                  :variant="plugin.enabled ? 'danger' : 'success'"
                  size="sm"
                >
                  <Power :size="16" />
                  {{ plugin.enabled ? i18n.t("config.disable") : i18n.t("config.enable") }}
                </SLButton>
                <SLButton @click.stop="deletePlugin(plugin)" variant="danger" size="sm">
                  <Trash2 :size="16" />
                  {{ i18n.t("config.delete") }}
                </SLButton>
              </div>
            </div>
          </div>
        </div>
      </template>
    </template>
  </div>
</template>

<style scoped>
.config-view {
  display: flex;
  flex-direction: column;
  gap: var(--sl-space-md);
}
.config-header {
  display: flex;
  flex-direction: column;
  gap: var(--sl-space-sm);
}
.server-path-display {
  color: var(--sl-text-tertiary);
  font-size: 0.75rem;
}
.empty-state {
  display: flex;
  align-items: center;
  justify-content: center;
  padding: var(--sl-space-2xl);
}
.error-banner,
.success-banner {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 10px 16px;
  border-radius: var(--sl-radius-md);
  font-size: 0.875rem;
}
.error-banner {
  background: rgba(239, 68, 68, 0.1);
  border: 1px solid rgba(239, 68, 68, 0.2);
  color: var(--sl-error);
}
.success-banner {
  background: rgba(34, 197, 94, 0.1);
  border: 1px solid rgba(34, 197, 94, 0.2);
  color: var(--sl-success);
}
.banner-close {
  font-weight: 600;
  background: none;
  border: none;
  cursor: pointer;
  color: inherit;
}
.loading-state {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: var(--sl-space-sm);
  padding: var(--sl-space-2xl);
  color: var(--sl-text-tertiary);
}
.config-entries {
  display: flex;
  flex-direction: column;
  gap: var(--sl-space-sm);
}
.config-entry {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: var(--sl-space-md);
  gap: var(--sl-space-lg);
  background: var(--sl-surface);
  border: 1px solid var(--sl-border-light);
  border-radius: var(--sl-radius-md);
  transition: all var(--sl-transition-fast);
}
.config-entry:hover {
  border-color: var(--sl-border);
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.05);
}
.entry-header {
  flex: 1;
  min-width: 0;
}
.entry-key-row {
  display: flex;
  align-items: center;
  gap: var(--sl-space-sm);
}
.entry-key {
  font-size: 0.875rem;
  font-weight: 600;
  color: var(--sl-text-primary);
}
.entry-desc {
  margin-top: 2px;
}
.entry-control {
  flex-shrink: 0;
  min-width: 200px;
}

.input {
  width: 200px;
  padding: 6px 10px;
  border: 1px solid var(--sl-border);
  border-radius: var(--sl-radius-sm);
  background: var(--sl-bg-secondary);
  color: var(--sl-text-primary);
}
.input:focus {
  outline: none;
  border-color: var(--sl-primary);
  box-shadow: 0 0 0 2px rgba(59, 130, 246, 0.1);
}

.integer-input {
  -moz-appearance: textfield;
}

.integer-input::-webkit-outer-spin-button,
.integer-input::-webkit-inner-spin-button {
  -webkit-appearance: none;
  margin: 0;
}

.tab-switcher {
  display: flex;
  gap: 8px;
  margin-top: 8px;
}

.tab-button {
  padding: 8px 16px;
  border-radius: var(--sl-radius-sm);
  font-size: 0.875rem;
  font-weight: 500;
  color: var(--sl-text-secondary);
  background: var(--sl-surface);
  border: 1px solid var(--sl-border);
  cursor: pointer;
  transition: all var(--sl-transition-fast);
}

.tab-button:hover {
  color: var(--sl-text-primary);
  border-color: var(--sl-border);
}

.tab-button.active {
  color: var(--sl-primary);
  background: var(--sl-primary-bg);
  border-color: var(--sl-primary);
}

.plugins-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 16px;
}

.plugins-header h3 {
  margin: 0;
  font-size: 1.125rem;
  font-weight: 600;
  color: var(--sl-text-primary);
}

.plugins-header-actions {
  display: flex;
  gap: 8px;
  align-items: center;
}

.reload-btn {
  font-size: 0.8125rem;
  padding: 6px 12px;
}

.reload-btn:hover {
  animation: shake 0.5s cubic-bezier(0.36, 0.07, 0.19, 0.97) both;
}

@keyframes shake {
  10%,
  90% {
    transform: translate3d(-1px, 0, 0);
  }
  20%,
  80% {
    transform: translate3d(2px, 0, 0);
  }
  30%,
  50%,
  70% {
    transform: translate3d(-4px, 0, 0);
  }
  40%,
  60% {
    transform: translate3d(4px, 0, 0);
  }
}

.plugins-container {
  display: flex;
  flex-direction: column;
  gap: 16px;
}
</style>
