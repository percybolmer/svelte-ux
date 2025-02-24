<script lang="ts">
  import type { SVGAttributes } from 'svelte/elements';

  import { uniqueId } from '$lib/utils/string';
  import { cls } from '$lib/utils/styles';
  import { getComponentTheme } from './theme';

  /** Apply gaussian blur.  Required unless blurring externally (`filter: blur()`, etc) */
  export let blur: number | undefined = undefined;

  /** `a4` of feColorMatrix.  See https://developer.mozilla.org/en-US/docs/Web/SVG/Element/feColorMatrix for details  */
  export let alphaPixel = 255;

  /** `a5` of feColorMatrix.  See https://developer.mozilla.org/en-US/docs/Web/SVG/Element/feColorMatrix for details  */
  export let alphaShift = -140;

  export let composite: SVGAttributes<SVGFECompositeElement>['operator'] = undefined;

  /** @type {{root?: string, icon?: string, loading?: string}} */
  export let classes: {
    root?: string;
    svg?: string;
  } = {};
  const theme = getComponentTheme('Gooey');

  const filterId = uniqueId('filter-');
</script>

<svg class={cls('fixed inset-0', theme.svg, classes?.svg)}>
  <filter id={filterId}>
    {#if blur}
      <feGaussianBlur in="SourceGraphic" stdDeviation={blur} result="blur" />
    {/if}

    <feColorMatrix
      in="blur"
      type="matrix"
      values="1 0 0 0 0
              0 1 0 0 0
              0 0 1 0 0
              0 0 0 {alphaPixel} {alphaShift}"
      result="goo"
    />

    {#if composite}
      <feComposite in="SourceGraphic" in2="goo" operator={composite} />
    {/if}
  </filter>
</svg>

<div
  style:filter="url(#{filterId})"
  {...$$restProps}
  class={cls('inline-block', theme.root, classes?.root, $$props.class)}
>
  <slot />
</div>
