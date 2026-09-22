<script setup lang="ts">
import Tooltip from './Tooltip.vue'
import type { ButtonProps } from './types.ts'

withDefaults(defineProps<ButtonProps>(), {
  type: 'primary',
  size: 'normal',
})

defineEmits(['click'])
</script>

<template>
  <Tooltip :disabled="!keyboard" :title="`${keyboard}`">
    <button
      type="button"
      class="base-button"
      v-bind="$attrs"
      @click="e => !disabled && !loading && $emit('click', e)"
      :class="[active && 'active', size, type, (disabled || loading) && 'disabled']"
      :disabled="disabled || loading"
      :aria-busy="loading"
    >
      <span :style="{ opacity: loading ? 0 : 1 }"><slot></slot></span>
      <IconEosIconsLoading v-if="loading" class="loading" width="18" :color="type === 'info' ? '#000000' : '#ffffff'" />
    </button>
  </Tooltip>
</template>

<style>
:root {
  --btn-primary: rgb(75, 85, 99);
  --btn-primary-disabled: #90969e;
  --btn-primary-hover: rgb(105, 121, 143);
  --btn-info: white;
  --btn-info-hover: #eaeaea;
  --btn-orange: #facc15;
  --btn-orange-hover: #bfac61;
}

html.dark {
  --btn-info: #1b1b1b;
  --btn-info-hover: #3a3a3a;
}
</style>

<style scoped lang="scss">
.base-button {
  position: relative;
  cursor: pointer;
  box-sizing: border-box;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  border: 1px solid transparent;
  text-align: center;
  transition:
    color var(--focus-duration) var(--focus-ease),
    background-color var(--focus-duration) var(--focus-ease),
    border-color var(--focus-duration) var(--focus-ease),
    box-shadow var(--focus-duration) var(--focus-ease);
  user-select: none;
  vertical-align: middle;
  white-space: nowrap;
  border-radius: var(--focus-radius-sm);
  padding: 0 1rem;
  font: inherit;
  font-size: 0.875rem;
  font-weight: 650;
  height: 2.5rem;
  color: white;

  & + .base-button {
    margin-left: 1rem;
  }

  &.disabled {
    opacity: 0.52;
    cursor: not-allowed;
    user-select: none;
    pointer-events: none;
    color: rgba(#fff, 0.72);
  }

  .loading {
    position: absolute;
  }

  &.small {
    border-radius: 0.5rem;
    padding: 0 0.75rem;
    min-height: 2.25rem;
    height: 2.25rem;
    font-size: 0.8125rem;
  }

  &.large {
    padding: 0 1.25rem;
    min-height: 2.875rem;
    height: 2.875rem;
    font-size: 0.9375rem;
    border-radius: 0.75rem;
  }

  & > span {
    line-height: 1;
    transform: translateY(-5%);

    :deep(a) {
      color: white;
    }
  }

  &.primary {
    background: var(--btn-primary);

    &.disabled {
      opacity: 1;
      background: var(--btn-primary-disabled);
    }

    &:hover:not(.disabled) {
      background: var(--btn-primary-hover);
      box-shadow: 0 6px 18px color-mix(in srgb, var(--focus-accent) 24%, transparent);
    }
  }

  &.info {
    background: var(--btn-info);
    border-color: var(--focus-border-strong);
    color: var(--color-main-text);

    &:hover:not(.disabled) {
      background: var(--btn-info-hover);
    }
  }

  &.text {
    border-color: var(--focus-border-strong);
    color: var(--color-main-text);

    &:hover:not(.disabled) {
      background: var(--btn-info);
    }
  }

  &.orange {
    background: var(--btn-orange);
    color: white;

    &:hover:not(.disabled) {
      background: var(--btn-orange-hover);
      color: white;
    }
  }

  &.active {
    opacity: 0.4;
  }
}
</style>
