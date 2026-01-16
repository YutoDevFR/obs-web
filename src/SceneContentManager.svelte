<script>
  import { onMount, onDestroy } from 'svelte'
  import { obs, sendCommand } from './obs.js'
  import Icon from 'mdi-svelte'
  import { mdiEye, mdiEyeOff, mdiVolumeHigh, mdiVolumeOff } from '@mdi/js'

  export let currentScene = ''

  let sceneItems = []
  let audioInputs = []

  // Types de sources audio connus dans OBS
  const AUDIO_INPUT_KINDS = [
    'wasapi_input_capture',      // Windows audio input
    'wasapi_output_capture',     // Windows audio output
    'coreaudio_input_capture',   // macOS audio input
    'coreaudio_output_capture',  // macOS audio output
    'pulse_input_capture',       // Linux audio input
    'pulse_output_capture',      // Linux audio output
    'ffmpeg_source',             // Media source (peut avoir audio)
    'vlc_source',                // VLC source
    'browser_source'             // Browser source (peut avoir audio)
  ]

  // Quand la scène courante change, charger son contenu
  $: if (currentScene) loadSceneContent(currentScene)

  async function loadSceneContent(sceneName) {
    if (!sceneName) return

    const data = await sendCommand('GetSceneItemList', { sceneName })
    sceneItems = data.sceneItems || []

    // Récupérer les sources audio et leur état mute
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

  // Event listeners pour màj temps réel
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

{#if currentScene}
<div class="scene-content-manager">
  <h3 class="title is-5">Contenu de : {currentScene}</h3>

  {#if sceneItems.length > 0}
    <div class="box">
      <h4 class="title is-6">Sources visuelles</h4>
      <ul class="item-list">
        {#each sceneItems as item}
          <li class="item-row" class:disabled={!item.sceneItemEnabled}>
            <button class="icon-btn" on:click={() => toggleVisibility(item)}>
              <Icon path={item.sceneItemEnabled ? mdiEye : mdiEyeOff} />
            </button>
            <span class="item-name">{item.sourceName}</span>
          </li>
        {/each}
      </ul>
    </div>
  {/if}

  {#if audioInputs.length > 0}
    <div class="box">
      <h4 class="title is-6">Sources audio</h4>
      <ul class="item-list">
        {#each audioInputs as input}
          <li class="item-row" class:muted={input.muted}>
            <button class="icon-btn" on:click={() => toggleMute(input)}>
              <Icon path={input.muted ? mdiVolumeOff : mdiVolumeHigh} />
            </button>
            <span class="item-name">{input.sourceName}</span>
          </li>
        {/each}
      </ul>
    </div>
  {/if}
</div>
{/if}

<style>
  .scene-content-manager {
    margin-bottom: 2rem;
  }
  .item-list {
    list-style: none;
    padding: 0;
    margin: 0;
  }
  .item-row {
    display: flex;
    align-items: center;
    padding: 0.5rem;
    border-radius: 4px;
    transition: background-color 0.2s;
  }
  .item-row:hover {
    background-color: rgba(0, 0, 0, 0.05);
  }
  .item-row.disabled {
    opacity: 0.5;
  }
  .item-row.muted {
    opacity: 0.7;
  }
  .icon-btn {
    background: none;
    border: none;
    cursor: pointer;
    padding: 0.25rem;
    margin-right: 0.5rem;
    color: #3e8ed0;
  }
  .icon-btn:hover {
    color: #1d72aa;
  }
  .item-row.disabled .icon-btn,
  .item-row.muted .icon-btn {
    color: #999;
  }
  .item-name {
    flex: 1;
  }
</style>
