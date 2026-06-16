<script setup lang="ts">
import { ref, onMounted } from 'vue'
import { useRouter } from 'vue-router'
import { NButton, NEmpty, NPopconfirm, NSpin, useMessage } from 'naive-ui'
import { useI18n } from 'vue-i18n'
import { fetchDeletedSessions, restoreSession, permanentDeleteSession } from '@/api/hermes/sessions'
import { formatTimestampSeconds } from '@/shared/session-display'
import PageSidebarNav from '@/components/layout/PageSidebarNav.vue'
import type { SessionSummary } from '@/api/hermes/sessions'

const { t } = useI18n()
const message = useMessage()
const router = useRouter()

const loading = ref(false)
const sessions = ref<SessionSummary[]>([])

async function loadDeleted() {
  loading.value = true
  try {
    sessions.value = await fetchDeletedSessions()
  } catch (err) {
    console.error('[trash] load failed', err)
    message.error(t('trash.loadFailed'))
  } finally {
    loading.value = false
  }
}

async function handleRestore(id: string) {
  const ok = await restoreSession(id)
  if (ok) {
    message.success(t('trash.restoreSuccess'))
    sessions.value = sessions.value.filter(s => s.id !== id)
  } else {
    message.error(t('trash.restoreFailed'))
  }
}

async function handlePermanentDelete(id: string) {
  const ok = await permanentDeleteSession(id)
  if (ok) {
    message.success(t('trash.permanentDeleteSuccess'))
    sessions.value = sessions.value.filter(s => s.id !== id)
  } else {
    message.error(t('trash.permanentDeleteFailed'))
  }
}

function openNewChatPage() {
  void router.push({ name: 'hermes.chat' })
}

function getModelLabel(session: SessionSummary): string {
  const parts: string[] = []
  if (session.model) parts.push(session.model)
  if (session.provider) parts.push(session.provider)
  return parts.join(' · ') || '-'
}

onMounted(() => {
  void loadDeleted()
})
</script>

<template>
  <div class="trash-view">
    <aside class="trash-sidebar">
      <div class="page-sidebar-top">
        <PageSidebarNav
          active="history"
          :primary-label="t('chat.newChat')"
          hide-mode-switch
          @primary="openNewChatPage"
        />
        <div class="trash-sidebar-header">
          <span class="trash-sidebar-title">{{ t('trash.title') }}</span>
        </div>
      </div>
    </aside>

    <div class="trash-content">
      <NSpin :show="loading">
        <NEmpty v-if="!loading && sessions.length === 0" :description="t('trash.empty')" />

        <div v-else class="trash-list">
          <div v-for="session in sessions" :key="session.id" class="trash-item">
            <div class="trash-item-info">
              <div class="trash-item-title">{{ session.title || t('trash.untitled') }}</div>
              <div class="trash-item-meta">
                <span class="trash-item-model">{{ getModelLabel(session) }}</span>
                <span class="trash-item-time">{{ formatTimestampSeconds(session.last_active || session.started_at) }}</span>
              </div>
            </div>
            <div class="trash-item-actions">
              <NButton size="tiny" @click="handleRestore(session.id)">
                {{ t('trash.restore') }}
              </NButton>
              <NPopconfirm
                :positive-text="t('trash.confirmPermanentDelete')"
                :negative-text="t('common.cancel')"
                @positive-click="handlePermanentDelete(session.id)"
              >
                <template #trigger>
                  <NButton size="tiny" type="error" secondary>
                    {{ t('trash.permanentDelete') }}
                  </NButton>
                </template>
                {{ t('trash.permanentDeleteConfirm') }}
              </NPopconfirm>
            </div>
          </div>
        </div>
      </NSpin>
    </div>
  </div>
</template>

<style scoped lang="scss">
@use "@/styles/variables" as *;

.trash-view {
  display: flex;
  height: 100%;
}

.trash-sidebar {
  width: 280px;
  border-right: 1px solid $border-color;
  display: flex;
  flex-direction: column;
  flex-shrink: 0;
  background: $bg-card;
  overflow-y: auto;
}

.page-sidebar-top {
  display: flex;
  flex-direction: column;
  padding: 8px 12px;
}

.trash-sidebar-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 8px 12px 0;
}

.trash-sidebar-title {
  font-size: 14px;
  font-weight: 600;
  color: $text-primary;
}

.trash-content {
  flex: 1;
  min-width: 0;
  padding: 24px;
  overflow-y: auto;
}

.trash-list {
  display: flex;
  flex-direction: column;
  gap: 6px;
  max-width: 640px;
}

.trash-item {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 12px 16px;
  border-radius: $radius-sm;
  background: $bg-card;
  border: 1px solid $border-color;
  gap: 16px;
}

.trash-item-info {
  flex: 1;
  min-width: 0;
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.trash-item-title {
  font-size: 14px;
  font-weight: 500;
  color: $text-primary;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.trash-item-meta {
  display: flex;
  gap: 12px;
  font-size: 12px;
  color: $text-muted;
}

.trash-item-actions {
  display: flex;
  gap: 8px;
  flex-shrink: 0;
}
</style>
