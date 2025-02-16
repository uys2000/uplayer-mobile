<template>
  <div class="w-full h-full flex flex-col lg:flex-row flex-nowrap p-4 gap-4 overflow-hidden"
    :class="{ 'pb-12': appStore.adLoaded }">
    <div class="w-full max-h-full rounded-box overflow-hidden bg-black flex">
      <img src="/logo.jpg" alt="logo" class="max-h-full overflow-hidden m-auto">
    </div>
    <div class="w-full">
      <div class="flex flex-col gap-4 mb-4">
        <span class="text-xl">Settings</span>
        <div class="flex flex-col gap-2">
          <div>
            <span>M3U File URL:</span>
            <div class="join w-full items-center">
              <div class="input input-bordered w-full flex items-center gap-2 join-item">
                <input type="text" class="grow h-full" placeholder="https://url.com/" v-model="url" />
              </div>
              <div class="indicator">
                <button class="btn join-item" @click="getUrl">Get</button>
              </div>
            </div>
          </div>
          <div>
            <span>Easy Access Code</span>
            <div class="join w-full items-center">
              <div class="input input-bordered w-full flex items-center gap-2 join-item min-w-16">
                <input type="text" class="grow h-full min-w-0" placeholder="" v-model="code" />
              </div>
              <button class="btn join-item" @click="createAccessCode">Create</button>
              <button class="btn join-item" @click="getAccessCode">Get</button>
              <button class="btn join-item" @click="deleteAccessCode">Remove</button>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import { ACCESSCODE, PLAYLIST } from '@/constant';
import { deleteFile, writeFileObject } from '@/services/capacitor/filesystem';
import { get } from '@/services/capacitor/http';
import { getPrefence, removePrefence, setPrefence } from '@/services/capacitor/preferences';
import { deleteDocument, getDocument, increaseDocument, setDocument } from '@/services/firebase/firestore';
import { parse } from '@/services/parser';
import { encode } from '@/services/sqids';
import { useAppStore } from '@/stores/app';
import { usePlaylistStore } from '@/stores/playlist';
import type { UPlayListItem } from '@/types/playlist';

export default {
  data() {
    return {
      url: "",
      code: "",
      playlistStore: usePlaylistStore(),
      appStore: useAppStore()
    }
  },
  methods: {
    async getIndex() {
      let index = 0;
      const indexDoc = await getDocument(ACCESSCODE, "-index")
      if (indexDoc.exists()) index = indexDoc.data().index
      return index
    },
    async updateChannels(data: string) {
      const playlist = parse.uLog(data)
      await deleteFile.uLog(PLAYLIST).catch(() => { })
      await writeFileObject.uLog(PLAYLIST, playlist.items.map(i => ({ line: i.line, url: i.url, name: i.name, group: i.group.title, img: i.tvg.logo } as UPlayListItem)))
    },
    async getUrl() {
      this.appStore.toast = "info"
      this.appStore.toastLabel = "Loading..."
      setPrefence.uLog("url", this.url)
      const response = await get.uLog(this.url, {})
      if (response.status == 200) {
        await this.updateChannels(response.data)
      }
      this.appStore.toast = "success"
      this.appStore.toastLabel = "Loaded "
    },
    async createAccessCode() {
      setPrefence.uLog("url", this.url)
      const code = encode(await this.getIndex())
      await setDocument.uLog(ACCESSCODE, code, { url: this.url, timesamp: Date.now() })
      await increaseDocument.uLog(ACCESSCODE, "-index", "index").catch(() => {
        return setDocument.uLog(ACCESSCODE, "-index", { index: 1 })
      })
      this.code = code
      setPrefence.uLog("code", this.code)
    },
    async getAccessCode() {
      const urlRef = await getDocument.uLog(ACCESSCODE, this.code)
      if (urlRef.exists()) this.url = urlRef.data().url
    },
    async deleteAccessCode() {
      await deleteDocument.uLog(ACCESSCODE, this.code)
      removePrefence.uLog("code")
      this.code = ""
    }
  },
  async beforeMount() {
    const [url, code] = await Promise.all([getPrefence.uLog("url"), getPrefence.uLog("code")])
    console.log(url, code)
    this.url = url || ""
    this.code = code || ""
  }
}
</script>

<style>
.input:focus,
.input input:focus,
.input:focus-within {
  outline: none;
}
</style>