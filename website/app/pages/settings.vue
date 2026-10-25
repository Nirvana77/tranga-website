<template>
    <TrangaPage>
        <UPageSection title="Settings" :ui="{ container: 'py-2 sm:py-2 lg:py-2 gap-2' }">
            <template #description>
                <div v-if="settingsStatus === 'error'">
                    <p class="text-warning">Unable to connect to api.</p>
                    <p class="">NUXT_PUBLIC_OPEN_FETCH_API_BASE_URL: {{ $config.public.openFetch.api.baseURL }}</p>
                </div>
            </template>
            <UCard v-if="settings">
                <template #header>
                    <h1>Libraries</h1>
                </template>
                <template #footer>
                    <div class="flex flex-row gap-2">
                        <UButton icon="i-lucide-plus" class="w-fit" @click="addLibraryModal.open()">Add FileLibrary</UButton>
                        <UTooltip :text="komgaConnected ? 'Disconnect Komga' : 'Connect Komga'">
                            <UButton
                                :icon="komgaConnected ? 'i-lucide-unlink' : 'i-lucide-link'"
                                class="w-fit"
                                label="Komga"
                                @click="onKomgaClick" />
                        </UTooltip>
                        <UTooltip :text="kavitaConnected ? 'Disconnect Kavita' : 'Connect Kavita'">
                            <UButton
                                :icon="kavitaConnected ? 'i-lucide-unlink' : 'i-lucide-link'"
                                class="w-fit"
                                label="Kavita"
                                @click="onKavitaClick" />
                        </UTooltip>
                    </div>
                </template>
                <FileLibraries />
            </UCard>
            <UCard v-if="settings">
                <template #header>
                    <h1>Downloads &amp; tasks</h1>
                </template>
                <UFormField
                    label="Download language"
                    :description="`Chapters are downloaded in this language. Websites with it: ${languageConnectors.join(', ') || 'none'}.`"
                    help="Global search only searches websites that have this language. Chapters already found keep their language.">
                    <USelectMenu
                        :model-value="settings?.downloadLanguage ?? undefined"
                        :items="languages"
                        class="w-48"
                        @update:model-value="setDownloadLanguage" />
                </UFormField>
                <div class="flex flex-row flex-wrap gap-6 mt-4">
                    <UFormField label="Tasks at the same time" help="Downloads, chapter checks, covers… all count.">
                        <UInputNumber
                            :model-value="settings?.maxConcurrentWorkers"
                            :min="1"
                            :max="64"
                            class="w-32"
                            @update:model-value="(n?: number | null) => setLimit('MaxConcurrentWorkers', n)" />
                    </UFormField>
                    <UFormField label="Chapter downloads at the same time" help="Each one also takes a task slot.">
                        <UInputNumber
                            :model-value="settings?.maxConcurrentDownloads"
                            :min="1"
                            :max="64"
                            class="w-32"
                            @update:model-value="(n?: number | null) => setLimit('MaxConcurrentDownloads', n)" />
                    </UFormField>
                </div>
                <UFormField
                    label="Task order"
                    help="When all slots are busy, the kind of task higher in this list starts first."
                    class="mt-4">
                    <div class="flex flex-col gap-1 w-fit">
                        <div v-for="(category, index) in taskPriority" :key="category" class="flex flex-row items-center gap-2">
                            <span class="text-dimmed w-4 text-right">{{ index + 1 }}</span>
                            <span class="w-64">{{ categoryLabels[category] }}</span>
                            <UButton
                                icon="i-lucide-arrow-up"
                                size="xs"
                                variant="ghost"
                                :disabled="index === 0"
                                @click="moveCategory(index, -1)" />
                            <UButton
                                icon="i-lucide-arrow-down"
                                size="xs"
                                variant="ghost"
                                :disabled="index === taskPriority.length - 1"
                                @click="moveCategory(index, 1)" />
                        </div>
                    </div>
                </UFormField>
            </UCard>
            <UCard v-if="settings">
                <template #header>
                    <h1>Notifications</h1>
                </template>
                <NotificationConnectors />
                <template #footer>
                    <div class="flex flex-row gap-2">
                        <UButton icon="i-lucide-plus" class="w-fit" @click="addGotifyModal.open()">Add Gotify</UButton>
                        <UButton icon="i-lucide-plus" class="w-fit" @click="addNtfyModal.open()">Add Ntfy</UButton>
                        <UButton icon="i-lucide-plus" class="w-fit" @click="addPushoverModal.open()">Add Pushover</UButton>
                        <UButton icon="i-lucide-plus" class="w-fit" @click="addGenericConnectorModal.open()"
                            >Add Generic Notification Connector</UButton
                        >
                    </div>
                </template>
            </UCard>
            <UCard v-if="settings">
                <template #header>
                    <h1>Maintenance</h1>
                </template>
                <div class="flex gap-2">
                    <UButton icon="i-lucide-database" loading-auto class="w-fit mb-2" @click="cleanUpDatabase">Clean database</UButton>
                    <UButton icon="i-lucide-captions-off" loading-auto class="w-fit mb-2" @click="cleanUpActions">Clean actions</UButton>
                </div>
            </UCard>
            <UCard>
                <template #header>
                    <h1>Stats</h1>
                </template>
                <div class="flex flex-row flex-wrap gap-2">
                    <UBadge v-for="(value, name) in stats" :key="name" variant="outline" color="neutral">
                        {{ deCamel(name) }}: {{ value }}
                    </UBadge>
                </div>
            </UCard>
        </UPageSection>
    </TrangaPage>
</template>

<script setup lang="ts">
import {
    LazyAddLibraryModal,
    LazyGenericNotificationConnectorModal,
    LazyGotifyModal,
    LazyKavitaModal,
    LazyKomgaModal,
    LazyNtfyModal,
    LazyPushoverModal,
} from '#components';
import FileLibraries from '~/components/FileLibraries.vue';
import type { components } from '#open-fetch-schemas/api';
import { refreshNuxtData } from '#app';
const overlay = useOverlay();
const { $api } = useNuxtApp();

const addLibraryModal = overlay.create(LazyAddLibraryModal);
const komgaModal = overlay.create(LazyKomgaModal);
const kavitaModal = overlay.create(LazyKavitaModal);

const addGotifyModal = overlay.create(LazyGotifyModal);
const addNtfyModal = overlay.create(LazyNtfyModal);
const addPushoverModal = overlay.create(LazyPushoverModal);
const addGenericConnectorModal = overlay.create(LazyGenericNotificationConnectorModal);

const cleanUpDatabase = async () => {
    await useApi('/v2/Maintenance/CleanupNoDownloadManga', { method: 'POST' });
    await refreshNuxtData(FetchKeys.Manga.All);
};
const cleanUpActions = async () => {
    await useApi('/v2/Maintenance/CleanupActions', { method: 'POST' });
};

const { data: libraries } = useApi('/v2/LibraryConnector', { key: FetchKeys.Libraries.All });
const komgaConnected = computed(() => libraries.value?.find((l) => l.type === 'Komga'));
const onKomgaClick = async () => {
    if (!komgaConnected.value) {
        komgaModal.open();
    } else {
        await $api('/v2/LibraryConnector/{LibraryConnectorId}', {
            method: 'DELETE',
            path: { LibraryConnectorId: komgaConnected.value.key },
        });
        await refreshNuxtData(FetchKeys.Libraries.All);
    }
};
const kavitaConnected = computed(() => libraries.value?.find((l) => l.type === 'Kavita'));
const onKavitaClick = async () => {
    if (!kavitaConnected.value) {
        kavitaModal.open();
    } else {
        await $api('/v2/LibraryConnector/{LibraryConnectorId}', {
            method: 'DELETE',
            path: { LibraryConnectorId: kavitaConnected.value.key },
        });
        await refreshNuxtData(FetchKeys.Libraries.All);
    }
};

// Gate the cards on settings, not on the status: the status is 'pending' during every refresh,
// which unmounted the cards and made the page jump
const {
    data: settings,
    status: settingsStatus,
    refresh: refreshSettings,
} = useApi('/v2/Settings', { key: FetchKeys.Settings.All, server: false });

const { data: mangaConnectors } = useApi('/v2/MangaConnector', { key: FetchKeys.MangaConnector.All, server: false });
const enabledConnectors = computed(() => mangaConnectors.value?.filter((c) => c.enabled && c.name !== 'Global') ?? []);
// Languages the enabled websites offer, the most widely offered first
const languages = computed(() => {
    const count = new Map<string, number>();
    for (const connector of enabledConnectors.value)
        for (const language of connector.supportedLanguages) count.set(language, (count.get(language) ?? 0) + 1);
    return [...count.keys()].sort((a, b) => count.get(b)! - count.get(a)! || a.localeCompare(b));
});
const languageConnectors = computed(() =>
    enabledConnectors.value.filter((c) => c.supportedLanguages.includes(settings.value?.downloadLanguage ?? '')).map((c) => c.name)
);
type WorkerCategory = components['schemas']['WorkerCategory'];
const categoryLabels: Record<WorkerCategory, string> = {
    Downloads: 'Chapter downloads',
    ChapterChecks: 'Checking for new chapters',
    Covers: 'Covers',
    Metadata: 'Metadata',
    Maintenance: 'Notifications, libraries & cleanup',
};
const taskPriority = computed<WorkerCategory[]>(() => settings.value?.taskPriority ?? []);
const moveCategory = async (index: number, by: number) => {
    const order = [...taskPriority.value];
    const [category] = order.splice(index, 1);
    order.splice(index + by, 0, category!);
    await $api('/v2/Settings/TaskPriority', { method: 'PATCH', body: order });
    await refreshSettings();
};
const setLimit = async (setting: 'MaxConcurrentWorkers' | 'MaxConcurrentDownloads', amount?: number | null) => {
    if (!amount || amount < 1 || !settings.value) return;
    // Show the new value right away instead of after the reload
    if (setting === 'MaxConcurrentWorkers') settings.value.maxConcurrentWorkers = amount;
    else settings.value.maxConcurrentDownloads = amount;
    if (setting === 'MaxConcurrentWorkers')
        await $api('/v2/Settings/MaxConcurrentWorkers/{Amount}', { method: 'PATCH', path: { Amount: amount } });
    else await $api('/v2/Settings/MaxConcurrentDownloads/{Amount}', { method: 'PATCH', path: { Amount: amount } });
    await refreshSettings();
};

const setDownloadLanguage = async (language: string) => {
    await $api('/v2/Settings/DownloadLanguage/{Language}', { method: 'PATCH', path: { Language: language } });
    await refreshSettings();
};

const { data: stats } = useApi('/v2/Stats', { server: false });
const deCamel = (camel: string): string =>
    camel.replace(/([a-z])([A-Z])/g, '$1 $2').replace(/(^\w{1})|(\s+\w{1})/g, (letter) => letter.toUpperCase());

useHead({ title: 'Settings' });
</script>
