<template>
  <form class="playlist-modal import-modal" @submit="submit">
    <label for="import-format">Format</label>
    <br />
    <select id="import-format" v-model="format" name="format" class="rounded-sm">
      <option value="library.xml">iTunes / Apple Music (Library.xml)</option>
    </select>

    <label for="import-file">File</label>
    <br />
    <input
      id="import-file"
      ref="fileInput"
      type="file"
      name="file"
      accept=".xml,application/xml,text/xml"
      class="rounded-sm"
      required
    />

    <p class="hint">
      iTunes stores absolute paths from the machine that exported the file. If
      that path doesn't match where Swing Music sees your music, fill in both
      fields below to rewrite it (e.g. <code>/Users/me/Music</code> →
      <code>/music</code>). Leave blank if the paths already match.
    </p>

    <label for="apple-prefix">Path in the file</label>
    <br />
    <input
      id="apple-prefix"
      v-model="applePrefix"
      type="text"
      name="apple_prefix"
      placeholder="/Users/me/Music"
      class="rounded-sm"
      spellcheck="false"
    />

    <label for="swingmusic-prefix">Path on this server</label>
    <br />
    <input
      id="swingmusic-prefix"
      v-model="swingmusicPrefix"
      type="text"
      name="swingmusic_prefix"
      placeholder="/music"
      class="rounded-sm"
      spellcheck="false"
    />

    <button type="submit" :disabled="busy">{{ busy ? "Importing..." : "Import" }}</button>
  </form>
</template>

<script setup lang="ts">
import { onMounted, ref } from "vue";

import { importPlaylists } from "@/requests/playlists";
import { NotifType, Notification } from "@/stores/notification";
import usePlaylistStore from "@/stores/pages/playlists";

const store = usePlaylistStore();

const format = ref("library.xml");
const applePrefix = ref("");
const swingmusicPrefix = ref("");
const busy = ref(false);

const emit = defineEmits<{
  (e: "setTitle", title: string): void;
  (e: "hideModal"): void;
}>();

emit("setTitle", "Import playlists");

onMounted(() => {
  document.getElementById("import-file")?.focus();
});

async function submit(e: Event) {
  e.preventDefault();
  if (busy.value) return;

  const form = e.target as HTMLFormElement;
  const file = (form.elements.namedItem("file") as HTMLInputElement).files?.[0];

  if (!file) {
    new Notification("Pick a file to import", NotifType.Error);
    return;
  }

  const apple = applePrefix.value.trim();
  const swing = swingmusicPrefix.value.trim();

  if (apple && !swing) {
    new Notification(
      "Set both prefix fields, or leave both blank",
      NotifType.Error
    );
    return;
  }

  const data = new FormData();
  data.set("file", file);
  data.set("format", format.value);
  if (apple) {
    data.set("apple_prefix", apple);
    data.set("swingmusic_prefix", swing);
  }

  busy.value = true;
  const result = await importPlaylists(data);
  busy.value = false;

  if (!result) return;

  await store.fetchAll();

  const created = result.created.length;
  const skipped = result.skipped.length;
  const parts = [`Imported ${created} playlist${created === 1 ? "" : "s"}`];
  if (skipped) parts.push(`${skipped} skipped`);
  if (result.unmatched) parts.push(`${result.unmatched} tracks unmatched`);

  new Notification(parts.join(", "), created ? NotifType.Success : NotifType.Info);
  emit("hideModal");
}
</script>

<style lang="scss">
.import-modal {
  display: grid;
  gap: 0.75rem;

  label {
    font-size: 0.9rem;
    font-weight: 500;
    color: $gray1;
  }

  input[type="text"],
  select {
    width: 100%;
    padding: $small $medium;
    background-color: $gray5;
    color: #fff;
    border: none;
    outline: none;
    font-size: 14px;
    height: 2.75rem;
  }

  input[type="file"] {
    color: $gray1;
    font-size: 0.9rem;
  }

  .hint {
    margin: 0;
    font-size: 0.85rem;
    color: $gray1;
    line-height: 1.4;
  }

  button {
    margin: 0.5rem auto 0;
    width: 8rem;
    padding: 1.25rem;
    background-color: $white;
    color: $black;

    &:hover {
      color: $white;
    }

    &:disabled {
      opacity: 0.6;
      cursor: not-allowed;
    }
  }
}
</style>
