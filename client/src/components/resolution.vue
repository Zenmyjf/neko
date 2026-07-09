<template>
  <vue-context class="context" ref="context">
    <li class="custom-resolution" @click.stop>
      <div class="custom-inputs">
        <input type="number" v-model.number="customWidth" min="240" max="7680" placeholder="Width" />
        <span>x</span>
        <input type="number" v-model.number="customHeight" min="240" max="4320" placeholder="Height" />
        <span>@</span>
        <input type="number" v-model.number="customRate" min="10" max="144" placeholder="Hz" />
      </div>
      <button class="custom-apply" @click.stop="applyCustom">Apply</button>
    </li>
    <li class="context-divider"></li>
    <template v-for="(conf, i) in configurations">
      <li
        :key="i"
        @click="screenSet(conf)"
        :class="[conf.width === width && conf.height === height && conf.rate === rate ? 'active' : '']"
      >
        <i class="fas fa-desktop"></i>
        <span>{{ conf.width }}x{{ conf.height }}</span>
        <small>{{ conf.rate }}</small>
      </li>
    </template>
  </vue-context>
</template>

<style lang="scss" scoped>
  .context {
    background-color: $background-floating;
    background-clip: padding-box;
    border-radius: 0.25rem;
    display: block;
    margin: 0;
    padding: 5px;
    min-width: 150px;
    z-index: 1500;
    position: fixed;
    list-style: none;
    box-sizing: border-box;
    max-height: calc(100% - 50px);
    overflow-y: auto;
    color: $interactive-normal;
    user-select: none;
    box-shadow: $elevation-high;
    scrollbar-width: thin;
    scrollbar-color: $background-secondary transparent;

    &::-webkit-scrollbar {
      width: 8px;
    }

    &::-webkit-scrollbar-track {
      background-color: transparent;
    }

    &::-webkit-scrollbar-thumb {
      background-color: $background-secondary;
      border: 2px solid $background-floating;
      border-radius: 4px;
    }

    &::-webkit-scrollbar-thumb:hover {
      background-color: $background-floating;
    }

    .custom-resolution {
      display: flex !important;
      flex-direction: column;
      align-items: stretch !important;
      cursor: default !important;
      padding: 8px !important;

      &:hover {
        background-color: transparent !important;
      }

      .custom-inputs {
        display: flex;
        align-items: center;
        gap: 4px;
        margin-bottom: 6px;

        input {
          width: 0;
          flex: 1 1 auto;
          background: $background-secondary;
          border: 1px solid $background-modifier-accent;
          border-radius: 3px;
          color: $interactive-hover;
          font-size: 0.85em;
          padding: 4px;
          text-align: center;

          &::-webkit-outer-spin-button,
          &::-webkit-inner-spin-button {
            -webkit-appearance: none;
            margin: 0;
          }
        }

        span {
          flex: 0 0 auto;
          color: $text-muted;
          font-size: 0.8em;
        }
      }

      .custom-apply {
        background: $style-primary;
        border: none;
        border-radius: 3px;
        color: #fff;
        font-size: 0.85em;
        padding: 6px;
        cursor: pointer;

        &:hover {
          filter: brightness(1.1);
        }
      }
    }

    .context-divider {
      height: 1px !important;
      margin: 4px 0 !important;
      padding: 0 !important;
      background: $background-modifier-accent;
      list-style: none;
      cursor: default !important;

      &:hover {
        background: $background-modifier-accent !important;
      }
    }

    > li {
      margin: 0;
      position: relative;
      align-content: center;
      display: flex;
      flex-direction: row;
      padding: 8px;
      cursor: pointer;
      border-radius: 3px;

      i {
        margin-right: 10px;
      }

      span {
        flex-grow: 1;
      }

      small {
        font-size: 0.7em;
        justify-self: flex-end;
        align-self: flex-end;
      }

      &.active,
      &:hover,
      &:focus {
        text-decoration: none;
        background-color: $background-modifier-hover;
        color: $interactive-hover;
      }

      &:focus {
        outline: 0;
      }
    }

    &:focus {
      outline: 0;
    }
  }
</style>

<script lang="ts">
  import { Component, Ref, Vue } from 'vue-property-decorator'
  import { ScreenResolution } from '~/neko/types'

  // @ts-ignore
  import { VueContext } from 'vue-context'

  @Component({
    name: 'neko-resolution',
    components: {
      'vue-context': VueContext,
    },
  })
  export default class extends Vue {
    @Ref('context') readonly context!: VueContext

    customWidth = 1080
    customHeight = 1920
    customRate = 30

    applyCustom() {
      const w = Math.round(this.customWidth)
      const h = Math.round(this.customHeight)
      const r = Math.round(this.customRate)

      if (!w || !h || !r || w < 240 || h < 240) {
        return
      }

      this.$accessor.video.screenSet({ width: w, height: h, rate: r })
      this.context.close()
    }

    get width() {
      return this.$accessor.video.width
    }

    get height() {
      return this.$accessor.video.height
    }

    get rate() {
      return this.$accessor.video.rate
    }

    get configurations() {
      return this.$accessor.video.configurations
    }

    open(event: MouseEvent) {
      this.context.open(event)
    }

    screenSet(resolution: ScreenResolution) {
      this.$accessor.video.screenSet(resolution)
    }
  }
</script>
