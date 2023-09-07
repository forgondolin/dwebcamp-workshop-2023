<script lang="ts">
  import * as odd from '@oddjs/odd'
  import { onDestroy } from 'svelte'
  import FileUploadCard from '$routes/avatars/components/upload/FileUploadCard.svelte'

  import { type Avatar, checkInitialize, checkSaveAvatar, fileToUint8Array } from '$routes/avatars/lib/avatars'
  import { filesystemStore, themeStore } from '$src/stores'

  let fs: odd.FileSystem

  const unsubscribeFileSystemStore = filesystemStore.subscribe(fileSystem => {
    fs = fileSystem
  })

  async function initialize() {
    if (fs) {
      const path = odd.path.directory('public', 'avatars')
      const exists = await fs.exists(path)
      
      if (!exists) {
        await fs.mkdir(path)
        await fs.publish()
      }
      await checkInitialize(fs)
    }
  }

  async function saveAvatar(event: CustomEvent<{ avatars: Avatar[] }>) {
  const { avatars } = event.detail

  if (fs) {
    await Promise.all(avatars.map(async avatar => {
      const path = odd.path.file('public', 'avatars', `${avatar.name}`)
      const bytes = await fileToUint8Array(avatar.file)

      await fs.write(path, bytes)

      await checkSaveAvatar(fs, avatar.name)
    }))
  }

  await fs.publish()
}

  initialize()

  onDestroy(unsubscribeFileSystemStore)
</script>

<section class="min-h-[calc(100vh-190px)] overflow-hidden">
  <FileUploadCard on:save={saveAvatar} />
</section>
