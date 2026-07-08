<template>
  <div v-if="hosting && is_touch_device" class="touchpad-panel" :class="{ collapsed }">
    <div class="tp-header">
      <span class="tp-title">TRACKPAD</span>
      <div class="tp-header-actions">
        <span class="tp-speed" @click.stop.prevent="cycleSpeed">{{ sensitivity.toFixed(1) }}x</span>
        <i
          class="fas"
          :class="collapsed ? 'fa-expand-alt' : 'fa-compress-alt'"
          @click.stop.prevent="collapsed = !collapsed"
        />
      </div>
    </div>
    <div v-show="!collapsed" class="tp-body">
      <div
        ref="pad"
        class="tp-pad"
        @touchstart.stop.prevent="onPadTouchStart"
        @touchmove.stop.prevent="onPadTouchMove"
        @touchend.stop.prevent="onPadTouchEnd"
        @touchcancel.stop.prevent="onPadTouchEnd"
      >
        <i class="fas fa-hand-pointer tp-hint-icon" />
        <div class="tp-hint">Drag to move &middot; Tap to click</div>
        <div class="tp-hint-sub">Two fingers scroll &middot; Hold to drag</div>
      </div>
      <div class="tp-buttons">
        <button class="tp-btn" @click.stop.prevent="remoteClick(1)">
          <i class="fas fa-hand-pointer" /><span>Left</span>
        </button>
        <button class="tp-btn" @click.stop.prevent="remoteClick(3)">
          <i class="fas fa-mouse" /><span>Right</span>
        </button>
        <button
          class="tp-btn"
          @touchstart.stop.prevent="startScroll(-1)"
          @touchend.stop.prevent="stopScroll"
          @touchcancel.stop.prevent="stopScroll"
        >
          <i class="fas fa-chevron-up" /><span>Up</span>
        </button>
        <button
          class="tp-btn"
          @touchstart.stop.prevent="startScroll(1)"
          @touchend.stop.prevent="stopScroll"
          @touchcancel.stop.prevent="stopScroll"
        >
          <i class="fas fa-chevron-down" /><span>Down</span>
        </button>
        <button class="tp-btn" @click.stop.prevent="sendZoom(1)">
          <i class="fas fa-search-plus" /><span>Zoom +</span>
        </button>
        <button class="tp-btn" @click.stop.prevent="sendZoom(-1)">
          <i class="fas fa-search-minus" /><span>Zoom -</span>
        </button>
        <button class="tp-btn" :class="{ active: dragLock }" @click.stop.prevent="toggleDragLock">
          <i class="fas fa-arrows-alt" /><span>Drag</span>
        </button>
        <button class="tp-btn" @click.stop.prevent="$emit('open-keyboard')">
          <i class="fas fa-keyboard" /><span>Keys</span>
        </button>
      </div>
    </div>
  </div>
</template>

<style lang="scss" scoped>
  .touchpad-panel {
    background: $background-tertiary;
    border-top: 1px solid rgba($color: #fff, $alpha: 0.08);
    flex-shrink: 0;
    padding: 8px 12px 10px 12px;
    display: none;

    // only ever shown on touch devices (component itself is v-if guarded too,
    // this display:none / flex swap is just for the responsive breakpoint)
    @media only screen and (max-width: 1024px) {
      display: flex;
      flex-direction: column;
    }

    &.collapsed {
      padding-bottom: 8px;
    }

    .tp-header {
      display: flex;
      align-items: center;
      justify-content: space-between;
      margin-bottom: 8px;

      .tp-title {
        font-size: 11px;
        letter-spacing: 1px;
        font-weight: 600;
        color: rgba($color: #fff, $alpha: 0.45);
      }

      .tp-header-actions {
        display: flex;
        align-items: center;
        gap: 10px;

        .tp-speed {
          font-size: 12px;
          padding: 3px 8px;
          border-radius: 10px;
          background: rgba($color: #fff, $alpha: 0.1);
          color: rgba($color: #fff, $alpha: 0.7);
        }

        i {
          font-size: 14px;
          color: rgba($color: #fff, $alpha: 0.6);
          padding: 4px;
        }
      }
    }

    .tp-body {
      display: flex;
      gap: 10px;
      height: 130px;

      .tp-pad {
        flex: 1 1 auto;
        min-width: 0;
        border: 1px solid rgba($color: #fff, $alpha: 0.12);
        border-radius: 10px;
        background: rgba($color: #fff, $alpha: 0.03);
        display: flex;
        flex-direction: column;
        align-items: center;
        justify-content: center;
        text-align: center;
        padding: 0 10px;
        touch-action: none;
        -webkit-user-select: none;
        user-select: none;

        .tp-hint-icon {
          font-size: 20px;
          color: rgba($color: #fff, $alpha: 0.3);
          margin-bottom: 8px;
        }

        .tp-hint {
          font-size: 13px;
          font-weight: 600;
          color: rgba($color: #fff, $alpha: 0.55);
        }

        .tp-hint-sub {
          font-size: 11px;
          color: rgba($color: #fff, $alpha: 0.3);
          margin-top: 2px;
        }
      }

      .tp-buttons {
        flex: 0 0 auto;
        width: 132px;
        display: grid;
        grid-template-columns: repeat(2, 1fr);
        gap: 6px;

        .tp-btn {
          display: flex;
          flex-direction: column;
          align-items: center;
          justify-content: center;
          gap: 3px;
          border: none;
          border-radius: 8px;
          background: rgba($color: #fff, $alpha: 0.08);
          color: rgba($color: #fff, $alpha: 0.8);
          font-size: 10px;
          -webkit-user-select: none;
          user-select: none;

          i {
            font-size: 15px;
          }

          &:active {
            background: rgba($color: #fff, $alpha: 0.18);
          }

          &.active {
            background: rgba($color: $style-primary, $alpha: 0.6);
            color: #fff;
          }
        }
      }
    }
  }
</style>

<script lang="ts">
  import { Component, Vue } from 'vue-property-decorator'

  const CONTROL_L = 0xffe3
  const KEY_PLUS = 0x2b
  const KEY_MINUS = 0x2d

  @Component({ name: 'neko-touchpad' })
  export default class extends Vue {
    collapsed = false
    sensitivity = 1.7
    private speeds = [1.0, 1.3, 1.7, 2.2]

    cycleSpeed() {
      const i = this.speeds.indexOf(this.sensitivity)
      this.sensitivity = this.speeds[(i + 1) % this.speeds.length]
    }

    get hosting() {
      return this.$accessor.remote.hosting
    }

    get controlling() {
      return this.$accessor.remote.controlling
    }

    get implicitHosting() {
      return this.$accessor.remote.implicitHosting
    }

    get locked() {
      return this.$accessor.remote.locked
    }

    get scroll() {
      // reuse the same user scroll-speed setting the rest of the app uses,
      // so button-taps feel like a normal, gradual mouse-wheel notch
      return this.$accessor.settings.scroll
    }

    get scroll_invert() {
      return this.$accessor.settings.scroll_invert
    }

    get is_touch_device() {
      return (
        ('ontouchstart' in window || navigator.maxTouchPoints > 0) &&
        window.matchMedia('(pointer: coarse)').matches
      )
    }

    private touchX = 0
    private touchY = 0
    private touchReady = false
    private touchLastX = 0
    private touchLastY = 0
    private touchMoved = false
    private twoFingerActive = false
    private lastTwoFingerY = 0

    private holdTimer: number | null = null
    private holdDragging = false
    dragLock = false

    private scrollInterval: number | null = null

    readonly TAP_MOVE_THRESHOLD = 8
    readonly HOLD_TO_DRAG_MS = 400

    private ensureReady() {
      if (!this.touchReady) {
        const { w, h } = this.$accessor.video.resolution
        this.touchX = w / 2
        this.touchY = h / 2
        this.touchReady = true
      }
    }

    private moveCursor(dx: number, dy: number) {
      const { w, h } = this.$accessor.video.resolution
      this.touchX = Math.min(Math.max(this.touchX + dx * this.sensitivity, 0), w)
      this.touchY = Math.min(Math.max(this.touchY + dy * this.sensitivity, 0), h)
      this.$client.sendData('mousemove', { x: Math.round(this.touchX), y: Math.round(this.touchY) })
    }

    remoteClick(button: number) {
      if (!this.hosting || this.locked) return
      this.ensureReady()
      this.$client.sendData('mousemove', { x: Math.round(this.touchX), y: Math.round(this.touchY) })
      this.$client.sendData('mousedown', { key: button })
      window.setTimeout(() => this.$client.sendData('mouseup', { key: button }), 60)
    }

    toggleDragLock() {
      if (!this.hosting || this.locked) return
      this.ensureReady()
      this.dragLock = !this.dragLock
      if (this.dragLock) {
        this.$client.sendData('mousemove', { x: Math.round(this.touchX), y: Math.round(this.touchY) })
        this.$client.sendData('mousedown', { key: 1 })
      } else {
        this.$client.sendData('mouseup', { key: 1 })
      }
    }

    sendZoom(direction: number) {
      // direction: 1 = zoom in, -1 = zoom out.
      // Sends the actual Ctrl+Plus / Ctrl+Minus browser shortcut, which moves
      // exactly one predictable zoom step per press (unlike ctrl+wheel, which
      // can jump several steps at once depending on how it's interpreted).
      if (!this.hosting || this.locked) return
      const key = direction > 0 ? KEY_PLUS : KEY_MINUS
      this.$client.sendData('keydown', { key: CONTROL_L })
      this.$client.sendData('keydown', { key })
      this.$client.sendData('keyup', { key })
      window.setTimeout(() => {
        this.$client.sendData('keyup', { key: CONTROL_L })
      }, 50)
    }

    private sendWheel(x: number, y: number) {
      // clamp to the same range the rest of the app uses for scroll speed,
      // so a single tap/tick feels like one gradual mouse-wheel notch
      const clampedX = Math.min(Math.max(x, -this.scroll), this.scroll)
      const clampedY = Math.min(Math.max(y, -this.scroll), this.scroll)
      this.$client.sendData('wheel', { x: clampedX, y: clampedY })
    }

    startScroll(direction: number) {
      // direction: -1 = up, 1 = down
      if (!this.hosting || this.locked) return
      const fire = () => this.sendWheel(0, direction * this.scroll)
      fire()
      this.scrollInterval = window.setInterval(fire, 220)
    }

    stopScroll() {
      if (this.scrollInterval) {
        clearInterval(this.scrollInterval)
        this.scrollInterval = null
      }
    }

    private requestControlIfNeeded() {
      if (!this.controlling && this.implicitHosting) {
        this.$accessor.remote.request()
      }
    }

    onPadTouchStart(e: TouchEvent) {
      if (!this.hosting || this.locked) return
      this.requestControlIfNeeded()
      this.ensureReady()

      if (e.touches.length === 1) {
        const t = e.touches[0]
        this.touchLastX = t.clientX
        this.touchLastY = t.clientY
        this.touchMoved = false
        this.twoFingerActive = false

        this.holdTimer = window.setTimeout(() => {
          if (!this.touchMoved && !this.dragLock && !this.holdDragging) {
            this.holdDragging = true
            this.$client.sendData('mousedown', { key: 1 })
          }
          this.holdTimer = null
        }, this.HOLD_TO_DRAG_MS)
      } else if (e.touches.length === 2) {
        if (this.holdTimer) {
          clearTimeout(this.holdTimer)
          this.holdTimer = null
        }
        this.twoFingerActive = true
        const [t1, t2] = [e.touches[0], e.touches[1]]
        this.lastTwoFingerY = (t1.clientY + t2.clientY) / 2
      }
    }

    onPadTouchMove(e: TouchEvent) {
      if (!this.hosting || this.locked) return

      if (e.touches.length === 1 && !this.twoFingerActive) {
        const t = e.touches[0]
        const dx = t.clientX - this.touchLastX
        const dy = t.clientY - this.touchLastY

        if (!this.touchMoved && (Math.abs(dx) > this.TAP_MOVE_THRESHOLD || Math.abs(dy) > this.TAP_MOVE_THRESHOLD)) {
          this.touchMoved = true
          if (this.holdTimer) {
            clearTimeout(this.holdTimer)
            this.holdTimer = null
          }
        }

        if (this.touchMoved) {
          this.moveCursor(dx, dy)
          this.touchLastX = t.clientX
          this.touchLastY = t.clientY
        }
      } else if (e.touches.length === 2) {
        const [t1, t2] = [e.touches[0], e.touches[1]]
        const midY = (t1.clientY + t2.clientY) / 2
        const yDelta = midY - this.lastTwoFingerY

        // small threshold avoids sending a flood of tiny scroll events
        if (Math.abs(yDelta) > 6) {
          const direction = this.scroll_invert ? -Math.sign(yDelta) : Math.sign(yDelta)
          this.sendWheel(0, direction * this.scroll)
          this.lastTwoFingerY = midY
        }
      }
    }

    onPadTouchEnd(e: TouchEvent) {
      if (!this.hosting || this.locked) return

      if (this.holdTimer) {
        clearTimeout(this.holdTimer)
        this.holdTimer = null
      }

      if (e.touches.length === 0) {
        if (this.holdDragging) {
          this.$client.sendData('mouseup', { key: 1 })
          this.holdDragging = false
        } else if (!this.touchMoved && !this.twoFingerActive && !this.dragLock) {
          this.remoteClick(1)
        }
        this.twoFingerActive = false
      }
    }
  }
</script>
