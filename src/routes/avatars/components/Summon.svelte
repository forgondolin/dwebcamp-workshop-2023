<script lang="ts">
  import * as odd from '@oddjs/odd'
  import { onDestroy } from 'svelte'
  import clipboardCopy from 'clipboard-copy'

  import { checkDeleteAvatar, fileToUint8Array, getAvatarsFromListing, getContentCID, type Avatar } from '$routes/avatars/lib/avatars'
  import { filesystemStore, themeStore } from '$src/stores'
  import AvatarCard from '$routes/avatars/components/summon/AvatarCard.svelte'
  import AvatarModal from '$routes/avatars/components/summon/AvatarModal.svelte'

  let fs: odd.FileSystem
  let avatars: Avatar[] = []
  let cidQuery = ''
  let isModalOpen: boolean
  let selectedAvatar: Avatar = null

  const unsubscribeFileSystemStore = filesystemStore.subscribe(fileSystem => {
    fs = fileSystem
  })

  async function loadAvatars() {
  if (fs) {
    const path = odd.path.directory('public', 'avatars')
    const exists = await fs.exists(path)

    if (exists) {
      const listing = await fs.ls(path)
      avatars = await getAvatarsFromListing(listing, fs)
    }
  }
}

async function deleteAvatar(event: CustomEvent<{ avatar: Avatar }>) {
  const { avatar } = event.detail

  if (fs) {
    const path = odd.path.file('public', 'avatars', `${avatar.name}`)

    await fs.rm(path)
    await fs.publish()

    loadAvatars()

    await checkDeleteAvatar(fs, avatar.name)
  }
}

  async function copyCID(event: CustomEvent<{ fileName: string }>) {
    const { fileName } = event.detail

    if (fs) {
      /**
       * Content identifiers (CIDs) are labels that point to content on
       * IPFS using content addressing. Read the IPFS docs for more information
       * about CIDs: https://docs.ipfs.tech/concepts/content-addressing/
       */
      const cid = await getContentCID(fileName, fs)
      await clipboardCopy(cid)
    }
  }

  async function openLink(event: CustomEvent<{ fileName: string }>) {
  const { fileName } = event.detail

  if (fs) {
    const cid = await getContentCID(fileName, fs)
    const url = `https://ipfs.runfission.com/ipfs/${cid}/userland`

    window.open(url, '_newtab')
  }
}

async function addAvatar() {
  const response = await fetch(`https://ipfs.runfission.com/ipfs/${cidQuery}/userland`)
  const blob = await response.blob()
  const file = new File([ blob ], cidQuery )
  const extension = blob.type.split('/')[1]

  const path = odd.path.file('public', 'avatars', `${cidQuery}.${extension}`)
  const bytes = await fileToUint8Array(file) 
  await fs.write(path, bytes)
  await fs.publish()

  await loadAvatars()
}

  $: {
    // Open modal when an avatar is selected
    isModalOpen = selectedAvatar !== null
  }

  onDestroy(unsubscribeFileSystemStore)

  loadAvatars()
</script>

<section
  class="min-h-[calc(100vh-190px)] overflow-hidden {$themeStore.selectedTheme ===
  'light'
    ? 'text-gray-800'
    : 'text-gray-200'}"
>
  <div
    class="grid grid-flow-row grid-rows-[2rem_auto] gap-8 w-full min-h-[calc(100vh-190px)]"
  >
    <div class="grid grid-flow-col grid-cols-[1fr_4rem] gap-2">
      <input
        class="w-full p-2 bg-base-100 border-2 border-base-content rounded text-base-content"
        type="text"
        placeholder="Enter a CID"
        bind:value={cidQuery}
      />
      <button
        class="btn btn-primary"
        on:click|preventDefault={addAvatar}
        on:keypress|preventDefault={addAvatar}
      >
        Add
      </button>
    </div>

    <div
      class="grid grid-cols-2 lg:grid-cols-4 xl:lg:grid-cols-6 gap-4"
    >
      {#each avatars as avatar}
        <AvatarCard {avatar} openModal={() => (selectedAvatar = avatar)} />
      {/each}
    </div>
  </div>

  {#if selectedAvatar}
    <AvatarModal
      bind:isModalOpen
      {selectedAvatar}
      {avatars}
      on:close={() => (selectedAvatar = null)}
      on:copycid={copyCID}
      on:delete={deleteAvatar}
      on:openlink={openLink}
    />
  {/if}
</section>
