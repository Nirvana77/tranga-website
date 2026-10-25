<template>
    <UPopover :content="{ side: 'top', align: 'end' }">
        <button
            type="button"
            class="fixed bottom-0 inset-x-0 z-50 h-(--status-bar-height) flex items-center gap-2 px-4 text-sm text-left bg-elevated border-t border-default cursor-pointer">
            <UIcon :name="icon.name" :class="['shrink-0 size-4', icon.class]" />
            <span class="truncate">{{ summary }}</span>
            <UProgress
                v-if="current?.progress != null"
                :model-value="current.progress * 100"
                size="xs"
                class="w-32 shrink-0 max-sm:hidden" />
            <span class="ml-auto shrink-0 text-dimmed">
                {{ running.length }}<template v-if="settings"> / {{ settings.maxConcurrentWorkers }}</template> running ·
                {{ queued.length }} queued
            </span>
        </button>
        <template #content>
            <div class="p-3 w-[min(32rem,calc(100vw-2rem))] max-h-[60dvh] overflow-y-auto flex flex-col gap-3 text-sm">
                <p v-if="error" class="text-warning">Can't reach the API.</p>
                <section>
                    <h2 class="font-semibold mb-1">Running</h2>
                    <p v-if="running.length === 0" class="text-dimmed">Nothing right now.</p>
                    <div v-for="worker in running" :key="worker.key" class="mb-2">
                        <p class="truncate">{{ worker.description }}</p>
                        <p v-if="worker.progressDescription" class="text-dimmed">{{ worker.progressDescription }}</p>
                        <UProgress v-if="worker.progress != null" :model-value="worker.progress * 100" size="xs" />
                    </div>
                </section>
                <section v-if="queued.length > 0">
                    <h2 class="font-semibold mb-1">Queued ({{ queued.length }})</h2>
                    <p v-for="worker in queued.slice(0, maxQueuedShown)" :key="worker.key" class="truncate text-dimmed">
                        {{ worker.description }}
                    </p>
                    <p v-if="queued.length > maxQueuedShown" class="text-dimmed">and {{ queued.length - maxQueuedShown }} more</p>
                </section>
                <section>
                    <h2 class="font-semibold mb-1">Scheduled</h2>
                    <p v-for="worker in scheduled" :key="worker.key" class="truncate text-dimmed">{{ worker.description }}</p>
                </section>
            </div>
        </template>
    </UPopover>
</template>

<script setup lang="ts">
import type { components } from '#open-fetch-schemas/api';
type Worker = components['schemas']['Worker'];

const maxQueuedShown = 20;
const refreshIntervalMs = 3000;

const { data: workers, error, refresh } = useApi('/v2/Worker', { key: FetchKeys.Workers.All, server: false });
// Shared with the settings page, so a changed limit shows up here too
const { data: settings } = useApi('/v2/Settings', { key: FetchKeys.Settings.All, server: false });

let timer: ReturnType<typeof setInterval> | undefined;
onMounted(() => {
    timer = setInterval(() => {
        if (!document.hidden) refresh();
    }, refreshIntervalMs);
});
onBeforeUnmount(() => clearInterval(timer));

// Downloads first: they're what people wait for
const byImportance = (a: Worker, b: Worker) =>
    Number(b.description.startsWith('Download ')) - Number(a.description.startsWith('Download ')) ||
    a.description.localeCompare(b.description);

const running = computed(() => (workers.value ?? []).filter((w) => w.state === 'Running').sort(byImportance));
const queued = computed(() =>
    (workers.value ?? []).filter((w) => !w.periodic && (w.state === 'Created' || w.state === 'Waiting')).sort(byImportance)
);
const scheduled = computed(() =>
    (workers.value ?? []).filter((w) => w.periodic && w.state !== 'Running').sort((a, b) => a.description.localeCompare(b.description))
);
const current = computed(() => running.value[0]);

const summary = computed(() => {
    if (error.value) return "Can't reach the API";
    if (!current.value) return queued.value.length > 0 ? 'Waiting for a free worker' : 'Idle';
    const more = running.value.length > 1 ? ` (+${running.value.length - 1} more)` : '';
    const progress = current.value.progressDescription ? ` · ${current.value.progressDescription}` : '';
    return `${current.value.description}${progress}${more}`;
});

const icon = computed(() =>
    error.value
        ? { name: 'i-lucide-circle-alert', class: 'text-warning' }
        : running.value.length > 0
          ? { name: 'i-lucide-loader-circle', class: 'animate-spin text-primary' }
          : { name: 'i-lucide-circle-check', class: 'text-success' }
);
</script>
