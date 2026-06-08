<script setup lang="ts">
import "./main.css"
import { useTemplateRef, ref, type Ref } from "vue";

const dropZone = useTemplateRef("dropZone");

window.addEventListener("drop", (e: DragEvent) => {
    if ([...e.dataTransfer!.items].some((item) => item.kind === "file")) {
        e.preventDefault();
    }
});

window.addEventListener("dragover", (e) => {
    const fileItems = [...e.dataTransfer!.items].filter(
        (item) => item.kind === "file",
    );
    if (fileItems.length > 0) {
        e.preventDefault();
        if (!dropZone.value!.contains(e.target as Node)) {
            e.dataTransfer!.dropEffect = "none";
        }
    }
});

const jsonl: Ref<{
    attributes: {
        level: string;
        exception_message?: string;
        backtrace?: string;
        named_tags: Record<string, string>,
    };
    message: string;
}[]> = ref([]);

function drop(ev: DragEvent) {
    ev.preventDefault();
    const file = ev.dataTransfer!.items[0]!.getAsFile()
    file!.text().then(data => jsonl.value = data.split("\n").slice(0, -1).map(v => JSON.parse(v)["attributes"]));
}

const shownTags = ["row_index", "current_url"];

const filters = ref(new Set(['info', 'error', 'warn']));

function toggleFilter(filter: string) {
    if (filters.value.has(filter)) filters.value.delete(filter);
    else filters.value.add(filter);
}
</script>

<template>
    <div v-if="!jsonl.length">
        <h2>Drop File:</h2>
        <div class="w-100 h-100 bg-gray-300" @drop="drop" ref="dropZone"></div>
    </div>
    <div v-else>
        <div class="flex gap-2 p-2 items-center">
            <div class="filter" :class="{selected: filters.has('info')}" @click="toggleFilter('info')">Info</div>
            <div class="filter" :class="{selected: filters.has('warn')}" @click="toggleFilter('warn')">Warn</div>
            <div class="filter" :class="{selected: filters.has('error')}" @click="toggleFilter('error')">Error</div>
            <span>Duplicate Jobs: {{jsonl.filter(l => l.message == "crawler.crawler.duplicate_job").length}}</span>
        </div>
        <div v-for="line in jsonl.filter(l => filters.has(l.attributes.level))" class="p-2 even:bg-gray-100 whitespace-pre-wrap">
            <span class="rounded-full p-1" :class="{
                'bg-green-500': line.attributes.level == 'info',
                'bg-red-500': line.attributes.level == 'error',
                'bg-yellow-500': line.attributes.level == 'warn',
            }">{{line.attributes.level}}</span> {{line.message.split("Backtrace")[0]}}
            <span v-if="line.attributes.exception_message" class="text-red-500">{{line.attributes.exception_message}}</span>
            <span v-if="line.attributes.backtrace" class="text-sm text-gray-500">{{ line.attributes.backtrace }}</span>
            <template v-for="tag of shownTags">
                <div v-if="tag in line.attributes?.named_tags" class="text-sm text-gray-500">
                    {{tag}}: <a :href="line.attributes.named_tags[tag]" target="_blank" v-if="tag == 'current_url'">{{line.attributes.named_tags[tag]}}</a>
                    <span v-else>{{line.attributes.named_tags[tag]}}</span>
                </div>
            </template>
        </div>
    </div>
</template>

<style scoped>
@reference "tailwindcss";
a {
    text-decoration: underline;
    @apply text-blue-300;
}
.filter {
    @apply rounded-full bg-gray-300 p-1 cursor-pointer;
    &.selected {
        @apply bg-blue-300;
    }
}
</style>
