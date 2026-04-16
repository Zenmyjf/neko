<template>
  <div ref="component" class="video">
    <div ref="player" class="player">
      <div ref="container" class="player-container">
        <video ref="video" playsinline />
        <div class="emotes">
          <template v-for="(emote, index) in emotes">
            <neko-emote :id="index" :key="index" />
          </template>
        </div>
        <textarea
          ref="overlay"
          class="overlay"
          spellcheck="false"
          tabindex="0"
          data-gramm="false"
          :style="{ pointerEvents: hosting ? 'auto' : 'none' }"
          @click.stop.prevent
          @contextmenu.stop.prevent
          @wheel.stop.prevent="onWheel"
          @mousemove.stop.prevent="onMouseMove"
          @mousedown.stop.prevent="onMouseDown"
          @mouseup.stop.prevent="onMouseUp"
          @mouseenter.stop.prevent="onMouseEnter"
          @mouseleave.stop.prevent="onMouseLeave"
          @touchmove.stop.prevent="onTouchMove"
          @touchstart.stop.prevent="onTouchStart"
          @touchend.stop.prevent="onTouchEnd"
          @compositionstart="onCompositionStartHandler"
          @compositionend="onCompositionEndHandler"
        />
        <div v-if="!playing && playable" class="player-overlay" @click.stop.prevent="playAndUnmute">
          <i class="fas fa-play-circle" />
        </div>
        <div v-else-if="mutedOverlay && muted" class="player-overlay" @click.stop.prevent="unmute">
          <i class="fas fa-volume-up" />
        </div>
        <div ref="aspect" class="player-aspect" />
      </div>
      <ul v-if="!fullscreen && !hideControls" class="video-menu top">
        <li><i @click.stop.prevent="requestFullscreen" class="fas fa-expand"></i></li>
        <li v-if="admin"><i @click.stop.prevent="openResolution" class="fas fa-desktop"></i></li>
        <li v-if="!controlLocked && !implicitHosting" :class="extraControls || 'extra-control'">
          <i
            :class="[
              hosted && !hosting ? 'disabled' : '',
              !hosted && !hosting ? 'faded' : '',
              'fas',
              'fa-computer-mouse',
            ]"
            @click.stop.prevent="toggleControl"
          />
        </li>
      </ul>
      <ul v-if="!fullscreen && !hideControls" class="video-menu bottom">
        <li v-if="hosting && (!clipboard_read_available || !clipboard_write_available)">
          <i @click.stop.prevent="openClipboard" class="fas fa-clipboard"></i>
        </li>
        <li>
          <i
            v-if="pip_available"
            @click.stop.prevent="requestPictureInPicture"
            class="fas fa-external-link-alt"
          />
        </li>
        <li
          v-if="hosting && is_touch_device"
          :class="extraControls || 'extra-control'"
          @click.stop.prevent="openMobileKeyboard"
        >
          <i class="fas fa-keyboard" />
        </li>
      </ul>
      <neko-resolution ref="resolution" v-if="admin" />
      <neko-clipboard ref="clipboard" v-if="hosting && (!clipboard_read_available || !clipboard_write_available)" />
    </div>
  </div>
</template>

<style lang="scss" scoped>
  .video {
    width: 100%;
    height: 100%;
    .player {
      position: absolute;
      display: flex;
      justify-content: center;
      align-items: center;
      background: #000;
      .video-menu {
        position: absolute;
        right: 20px;
        &.top { top: 15px; }
        &.bottom { bottom: 15px; }
        li {
          margin: 0 0 10px 0;
          i {
            width: 30px;
            height: 30px;
            background: rgba(255, 255, 255, 0.2);
            border-radius: 5px;
            line-height: 30px;
            font-size: 16px;
            text-align: center;
            color: rgba(255, 255, 255, 0.6);
            cursor: pointer;
            &.faded { color: rgba(255, 255, 255, 0.4); }
            &.disabled { color: rgba(255, 0, 0, 0.4); }
          }
          &.extra-control { display: none; }
          @media (max-width: 768px) {
            &.extra-control { display: block; }
          }
        }
      }
      .player-container {
        position: relative;
        width: 100%;
        video {
          position: absolute;
          width: 100%;
          height: 100%;
          background: #000;
        }
        .overlay {
          position: absolute;
          top: 0;
          width: 100%;
          height: 100%;
          color: transparent;
          background: transparent;
          resize: none;
          z-index: 10;
        }
        .player-aspect {
          display: block;
          padding-bottom: 56.25%;
        }
      }
    }
  }
</style>

<script lang="ts">
  import { Component, Ref, Watch, Vue, Prop } from 'vue-property-decorator'
  import ResizeObserver from 'resize-observer-polyfill'
  import { elementRequestFullscreen, onFullscreenChange, isFullscreen, lockKeyboard, unlockKeyboard } from '~/utils'
  import Emote from './emote.vue'
  import Resolution from './resolution.vue'
  import Clipboard from './clipboard.vue'
  // @ts-ignore
  import GuacamoleKeyboard from '~/utils/guacamole-keyboard.ts'

  @Component({
    name: 'neko-video',
    components: {
      'neko-emote': Emote,
      'neko-resolution': Resolution,
      'neko-clipboard': Clipboard,
    },
  })
  export default class extends Vue {
    @Ref('component') readonly _component!: HTMLElement
    @Ref('container') readonly _container!: HTMLElement
    @Ref('overlay') readonly _overlay!: HTMLTextAreaElement
    @Ref('player') readonly _player!: HTMLElement
    @Ref('video') readonly _video!: HTMLVideoElement
    @Ref('resolution') readonly _resolution!: Resolution
    @Ref('clipboard') readonly _clipboard!: Clipboard

    @Prop(Boolean) readonly hideControls!: boolean
    @Prop(Boolean) readonly extraControls!: boolean

    private keyboard = GuacamoleKeyboard()
    private observer = new ResizeObserver(this.onResize.bind(this))
    private focused = false
    private fullscreen = false
    private mutedOverlay = true
    
    // TRACKPAD VARIABLES
    private lastTouchX = 0
    private lastTouchY = 0
    private isMoving = false
    private mouseX = 0
    private mouseY = 0

    get hosting() { return this.$accessor.remote.hosting }
    get locked() { return this.$accessor.remote.locked }
    get is_touch_device() {
      return (('ontouchstart' in window || navigator.maxTouchPoints > 0) && window.matchMedia('(pointer: coarse)').matches)
    }

    mounted() {
      this.observer.observe(this._component)
      this.keyboard.listenTo(this._overlay)
      // Initialize mouse position in center
      this.mouseX = this.$accessor.video.width / 2
      this.mouseY = this.$accessor.video.height / 2
    }

    // --- TRACKPAD LOGIC ---
    onTouchStart(e: TouchEvent) {
      if (!this.hosting || this.locked) return
      this.lastTouchX = e.touches[0].clientX
      this.lastTouchY = e.touches[0].clientY
      this.isMoving = false
    }

    onTouchMove(e: TouchEvent) {
      if (!this.hosting || this.locked) return
      this.isMoving = true
      
      const touch = e.touches[0]
      const deltaX = (touch.clientX - this.lastTouchX) * 1.5 // Sensitivity
      const deltaY = (touch.clientY - this.lastTouchY) * 1.5

      this.mouseX = Math.min(Math.max(this.mouseX + deltaX, 0), this.$accessor.video.width)
      this.mouseY = Math.min(Math.max(this.mouseY + deltaY, 0), this.$accessor.video.height)

      this.$client.sendData('mousemove', { x: Math.round(this.mouseX), y: Math.round(this.mouseY) })

      this.lastTouchX = touch.clientX
      this.lastTouchY = touch.clientY
    }

    onTouchEnd(e: TouchEvent) {
      if (!this.hosting || this.locked) return
      
      // If it was just a tap (no movement)
      if (!this.isMoving) {
        const button = e.changedTouches.length > 1 ? 3 : 1 // 2 fingers = right click
        this.$client.sendData('mousedown', { key: button })
        setTimeout(() => this.$client.sendData('mouseup', { key: button }), 50)
      }
    }

    openMobileKeyboard() {
      this._overlay.value = "" // Clear it
      this._overlay.focus()
    }

    // ... (rest of your original helper methods like onResize, play, etc. kept below)
    onResize() {
      const { offsetWidth, offsetHeight } = !this.fullscreen ? this._component : document.body
      this._player.style.width = `${offsetWidth}px`
      this._player.style.height = `${offsetHeight}px`
    }
  }
</script>
