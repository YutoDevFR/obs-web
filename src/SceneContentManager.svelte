<script>
  import { onMount, onDestroy } from 'svelte'
  import { obs, sendCommand } from './obs.js'
  import Icon from 'mdi-svelte'
  import { mdiEye, mdiEyeOff, mdiVolumeHigh, mdiVolumeOff, mdiRefresh } from '@mdi/js'

  export let currentScene = ''

  let sceneItems = []
  let audioInputs = []

  // Types de sources audio connus dans OBS
  const AUDIO_INPUT_KINDS = [
    'wasapi_input_capture',
    'wasapi_output_capture',
    'coreaudio_input_capture',
    'coreaudio_output_capture',
    'pulse_input_capture',
    'pulse_output_capture',
    'ffmpeg_source',
    'vlc_source',
    'browser_source'
  ]

  // Types de sources refreshables
  const MEDIA_SOURCE_KINDS = ['ffmpeg_source', 'vlc_source']
  const BROWSER_SOURCE_KINDS = ['browser_source']

  $: if (currentScene) loadSceneContent(currentScene)

  async function loadSceneContent(sceneName) {
    if (!sceneName) return

    const data = await sendCommand('GetSceneItemList', { sceneName })
    sceneItems = data.sceneItems || []

    audioInputs = []
    for (const item of sceneItems) {
      if (AUDIO_INPUT_KINDS.includes(item.inputKind)) {
        const muteData = await sendCommand('GetInputMute', { inputName: item.sourceName })
        audioInputs = [...audioInputs, { ...item, muted: muteData.inputMuted }]
      }
    }
  }

  async function toggleVisibility(item) {
    await sendCommand('SetSceneItemEnabled', {
      sceneName: currentScene,
      sceneItemId: item.sceneItemId,
      sceneItemEnabled: !item.sceneItemEnabled
    })
  }

  async function toggleMute(input) {
    await sendCommand('SetInputMute', {
      inputName: input.sourceName,
      inputMuted: !input.muted
    })
  }

  async function refreshSource(item) {
    if (MEDIA_SOURCE_KINDS.includes(item.inputKind)) {
      // Restart media playback
      await sendCommand('TriggerMediaInputAction', {
        inputName: item.sourceName,
        mediaAction: 'OBS_WEBSOCKET_MEDIA_INPUT_ACTION_RESTART'
      })
    } else if (BROWSER_SOURCE_KINDS.includes(item.inputKind)) {
      // Refresh browser cache
      await sendCommand('PressInputPropertiesButton', {
        inputName: item.sourceName,
        propertyName: 'refreshnocache'
      })
    }
  }

  function isRefreshable(item) {
    return MEDIA_SOURCE_KINDS.includes(item.inputKind) || BROWSER_SOURCE_KINDS.includes(item.inputKind)
  }

  function onVisibilityChanged(data) {
    if (data.sceneName === currentScene) {
      sceneItems = sceneItems.map(item =>
        item.sceneItemId === data.sceneItemId
          ? { ...item, sceneItemEnabled: data.sceneItemEnabled }
          : item
      )
    }
  }

  function onMuteChanged(data) {
    audioInputs = audioInputs.map(input =>
      input.sourceName === data.inputName
        ? { ...input, muted: data.inputMuted }
        : input
    )
  }

  function onSceneItemCreated(data) {
    if (data.sceneName === currentScene) {
      loadSceneContent(currentScene)
    }
  }

  function onSceneItemRemoved(data) {
    if (data.sceneName === currentScene) {
      loadSceneContent(currentScene)
    }
  }

  onMount(() => {
    obs.on('SceneItemEnableStateChanged', onVisibilityChanged)
    obs.on('InputMuteStateChanged', onMuteChanged)
    obs.on('SceneItemCreated', onSceneItemCreated)
    obs.on('SceneItemRemoved', onSceneItemRemoved)
  })

  onDestroy(() => {
    obs.off('SceneItemEnableStateChanged', onVisibilityChanged)
    obs.off('InputMuteStateChanged', onMuteChanged)
    obs.off('SceneItemCreated', onSceneItemCreated)
    obs.off('SceneItemRemoved', onSceneItemRemoved)
  })
</script>

{#if currentScene && sceneItems.length > 0}
<div class="scene-content-manager">
  <div class="section-header">
    <span class="section-title">Sources</span>
    <span class="scene-name">{currentScene}</span>
  </div>

  <div class="items-grid">
    {#each sceneItems as item}
      <div class="item-row" class:has-refresh={isRefreshable(item)}>
        <button
          class="item-chip"
          class:disabled={!item.sceneItemEnabled}
          class:with-refresh={isRefreshable(item)}
          on:click={() => toggleVisibility(item)}
          title={item.sceneItemEnabled ? 'Cacher' : 'Afficher'}
        >
          <span class="icon"><Icon path={item.sceneItemEnabled ? mdiEye : mdiEyeOff} /></span>
          <span class="item-name">{item.sourceName}</span>
        </button>
        {#if isRefreshable(item)}
          <button class="refresh-btn" on:click|stopPropagation={() => refreshSource(item)} title="Rafraîchir">
            <Icon path={mdiRefresh} />
          </button>
        {/if}
      </div>
    {/each}
  </div>

  {#if audioInputs.length > 0}
    <div class="section-header audio-header">
      <span class="section-title">Audio</span>
    </div>
    <div class="items-grid">
      {#each audioInputs as input}
        <button
          class="item-chip audio-chip"
          class:muted={input.muted}
          on:click={() => toggleMute(input)}
          title={input.muted ? 'Unmute' : 'Mute'}
        >
          <span class="icon"><Icon path={input.muted ? mdiVolumeOff : mdiVolumeHigh} /></span>
          <span class="item-name">{input.sourceName}</span>
        </button>
      {/each}
    </div>
  {/if}
</div>
{/if}

<style>
  .scene-content-manager {
    margin-bottom: 1.5rem;
    background: #f5f5f5;
    border-radius: 8px;
    padding: 0.75rem;
  }
  .section-header {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    margin-bottom: 0.5rem;
  }
  .audio-header {
    margin-top: 0.75rem;
  }
  .section-title {
    font-weight: 600;
    font-size: 0.75rem;
    text-transform: uppercase;
    color: #666;
  }
  .scene-name {
    font-size: 0.75rem;
    color: #999;
  }
  .items-grid {
    display: flex;
    flex-wrap: wrap;
    gap: 0.35rem;
  }
  .item-row {
    display: inline-flex;
    align-items: stretch;
    border-radius: 4px;
    overflow: hidden;
  }
  .item-row.has-refresh {
    box-shadow: 0 1px 2px rgba(0,0,0,0.1);
  }
  .item-chip.with-refresh {
    border-top-right-radius: 0;
    border-bottom-right-radius: 0;
  }
  .refresh-btn {
    padding: 0 0.4rem;
    border-radius: 0 4px 4px 0;
    border: none;
    background: rgba(0,0,0,0.15);
    color: white;
    cursor: pointer;
    display: flex;
    align-items: center;
    transition: background 0.15s ease;
  }
  .refresh-btn:hover {
    background: rgba(0,0,0,0.25);
  }
  .refresh-btn:active {
    background: rgba(0,0,0,0.35);
  }
  .refresh-btn :global(svg) {
    width: 12px;
    height: 12px;
  }
  .item-row.has-refresh .item-chip {
    background: #3e8ed0;
  }
  .item-row.has-refresh .item-chip.disabled {
    background: #aaa;
  }
  .item-row.has-refresh .refresh-btn {
    background: #2d7fc4;
  }
  .item-row.has-refresh .refresh-btn:hover {
    background: #2570b0;
  }
  .item-row.has-refresh .item-chip.disabled + .refresh-btn {
    background: #999;
  }
  .item-row.has-refresh .item-chip.disabled + .refresh-btn:hover {
    background: #888;
  }
  .item-chip {
    display: inline-flex;
    align-items: center;
    gap: 0.3rem;
    padding: 0.25rem 0.5rem;
    border-radius: 4px;
    border: none;
    background: #3e8ed0;
    color: white;
    font-size: 0.7rem;
    cursor: pointer;
    transition: all 0.15s ease;
    max-width: 180px;
  }
  .item-chip:hover {
    filter: brightness(1.1);
  }
  .item-chip.disabled {
    background: #ccc;
    color: #666;
  }
  .item-chip.audio-chip {
    background: #48c78e;
  }
  .item-chip.audio-chip.muted {
    background: #ccc;
    color: #666;
  }
  .item-name {
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }
  .icon {
    display: flex;
    align-items: center;
    width: 14px;
    height: 14px;
    flex-shrink: 0;
  }
  .icon :global(svg) {
    width: 14px;
    height: 14px;
  }
</style>
