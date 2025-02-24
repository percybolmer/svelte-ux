<script lang="ts">
  import { createEventDispatcher, type ComponentProps } from 'svelte';
  import type { HTMLInputAttributes } from 'svelte/elements';
  import { mdiClose, mdiCurrencyUsd, mdiEye, mdiInformationOutline, mdiPercent } from '@mdi/js';
  import { uniqueId } from 'lodash-es';

  import { autoFocus } from '../actions/input';
  import { multi } from '../actions/multi';
  import type { Actions } from '../actions/multi';
  import { cls } from '../utils/styles';
  import { isLiteralObject } from '../utils/object';
  import { getComponentTheme } from './theme';

  import Button from './Button.svelte';
  import Icon from './Icon.svelte';
  import Input from './Input.svelte';

  type InputValue = string | number;

  const dispatch = createEventDispatcher<{
    clear: null;
    change: { value: typeof value; inputValue: InputValue; operator: string };
  }>();

  export let name: string | undefined = undefined;
  export let label = '';
  export let labelPlacement: 'inset' | 'float' | 'top' | 'left' = 'inset';
  export let value: InputValue | { [operator: string]: InputValue } = ''; // TODO: Can also include operator: { "operator": "value" }
  export let type:
    | 'text'
    | 'password'
    | 'integer'
    | 'decimal'
    | 'currency'
    | 'percent'
    | 'search'
    | 'email' = 'text';
  export let placeholder: string | undefined = undefined;
  export let error: string | string[] | boolean | undefined = '';
  export let hint = '';
  export let autocomplete = 'off'; // https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/autocomplete
  export let multiline = false;
  export let disabled = false;
  export let clearable = false;
  export let base = false;
  export let rounded = false;
  export let dense = false;
  export let icon: string | null = null;
  export let iconRight: string | null = null;
  export let align: 'left' | 'center' | 'right' = 'left';
  export let autofocus: boolean | Parameters<typeof autoFocus>[1] = false;
  // TODO: Find way to conditionally set type based on `multiline` value
  export let actions: Actions<HTMLInputElement | HTMLTextAreaElement> = autofocus
    ? (node) => [autoFocus(node, typeof autofocus === 'object' ? autofocus : undefined)]
    : undefined;
  export let operators: { label: string; value: string }[] = undefined;
  export let inputEl: HTMLInputElement | null = null;
  export let debounceChange: boolean | number = false;
  export let classes: {
    root?: string;
    container?: string;
    label?: string;
    input?: string;
    error?: string;
    prepend?: string;
    append?: string;
  } = {};
  const theme = getComponentTheme('TextField');

  // Input props
  export let mask: string | undefined = undefined;
  export let replace: string | undefined = undefined;
  export let accept: string | RegExp | undefined = undefined;
  // https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes/autocapitalize
  export let autocapitalize: ComponentProps<Input>['autocapitalize'] = undefined;

  let inputType = 'text';
  $: switch (type) {
    case 'integer':
    case 'decimal':
    case 'currency':
    case 'percent':
      // TODO: typing '.' on iOS appears to clear input when using type="number"
      inputType = 'number';
      break;
    case 'password':
      inputType = 'password';
      break;
    case 'email':
      inputType = 'email';
      break;
    case 'search':
      inputType = 'search';
      break;
    case 'text':
    default:
      inputType = 'text';
  }
  let inputMode: HTMLInputAttributes['inputmode'] = undefined;
  $: switch (type) {
    case 'integer':
      inputMode = 'numeric';
      break;
    case 'decimal':
    case 'currency':
    case 'percent':
      inputMode = 'decimal';
      break;
    case 'email':
      inputMode = 'email';
      break;
    case 'search':
      inputMode = 'search';
      break;
    case 'text':
    case 'password':
    default:
      inputMode = 'text';
  }

  $: inputValue = isLiteralObject(value) ? Object.values(value)[0] : value ?? null;
  $: operator = isLiteralObject(value) ? Object.keys(value)[0] : operators?.[0].value;

  let lastTimeoutId: ReturnType<typeof setTimeout>;
  function updateValue() {
    let newValue;
    // TODO: Improve handling of `1234.05` and backspacing the `5`, which results in `1234` (`.0` removed)
    const valueAsType = inputType === 'number' ? Number(inputValue) : inputValue;
    if (inputValue && operator) {
      // Add with operator if used
      newValue = { [operator]: valueAsType };
    } else {
      newValue = inputValue === '' ? null : valueAsType;
    }

    value = newValue;
    if (debounceChange) {
      clearTimeout(lastTimeoutId);
      lastTimeoutId = setTimeout(
        () => {
          dispatch('change', { value, inputValue, operator });
        },
        debounceChange === true ? 300 : debounceChange
      );
    } else {
      dispatch('change', { value, inputValue, operator });
    }
  }

  function handleInput(e) {
    if (accept) {
      // filter input based on accepted characters
      const regex = new RegExp(accept, 'g');
      inputValue = e.target.value.match(regex)?.[0] ?? '';
      e.target.value = inputValue;
    } else {
      inputValue = e.target.value;
    }
    updateValue();
  }

  $: hasInputValue = inputValue != null && inputValue !== '';
  $: hasInsetLabel = ['inset', 'float'].includes(labelPlacement) && label !== '';

  $: hasPrepend = $$slots.prepend || icon != null;
  $: hasAppend =
    $$slots.append || iconRight != null || clearable || error || operators || type === 'password';
  $: hasPrefix = $$slots.prefix || type === 'currency';
  $: hasSuffix = $$slots.suffix || type === 'percent';

  const id = uniqueId('textfield-');
  let labelEl: HTMLLabelElement | null = null;
</script>

<fieldset
  {disabled}
  class={cls(
    'TextField',
    'group flex gap-1',
    labelPlacement !== 'left' ? 'flex-col' : 'items-center',
    error ? '[--color:theme(colors.red.500)]' : '[--color:theme(colors.accent.500)]',
    disabled && 'opacity-50 pointer-events-none',
    !base && (rounded ? 'rounded-full' : 'rounded'),
    theme.root,
    classes.root,
    $$props.class
  )}
>
  {#if label && ['top', 'left'].includes(labelPlacement)}
    <label
      class={cls(
        'block text-sm font-medium',
        'truncate group-hover:text-gray-700 group-focus-within:text-[var(--color)] group-hover:group-focus-within:text-[var(--color)] cursor-pointer',
        error ? 'text-red-500/80' : 'text-black/50',
        `placement-${labelPlacement}`,
        theme.label,
        classes.label
      )}
      for={id}
      bind:this={labelEl}
    >
      {label}
    </label>
  {/if}

  <div class="flex-1">
    <div
      class={cls(
        'border py-0 transition-shadow',
        disabled ? '' : 'hover:shadow',
        disabled ? '' : error ? 'hover:border-red-700' : 'hover:border-gray-700',
        {
          'px-2': !rounded,
          'px-6': rounded && !hasPrepend, // TODO: `hasPrepend` always true for SelectField, etc.  See: https://github.com/sveltejs/svelte/issues/6059
        },
        !base && ['bg-white', rounded ? 'rounded-full' : 'rounded'],
        error ? 'border-red-500' : 'border-black/20',
        'group-focus-within:shadow-md group-focus-within:border-[var(--color)]',
        theme.container,
        classes.container
      )}
    >
      <div class="flex items-center">
        {#if hasPrepend}
          <div
            class={cls(
              'prepend whitespace-nowrap',
              rounded && 'pl-3',
              theme.prepend,
              classes.prepend
            )}
          >
            <slot name="prepend" />
            {#if icon}
              <span class="mr-3">
                <Icon path={icon} class="text-black/50" />
              </span>
            {/if}
          </div>
        {/if}

        <!-- svelte-ignore a11y-click-events-have-key-events -->
        <div class="flex-grow inline-grid" on:click>
          {#if label && ['inset', 'float'].includes(labelPlacement)}
            <label
              class={cls(
                'col-span-full row-span-full z-[1] flex items-center h-full truncate origin-top-left transition-all duration-200 group-hover:text-gray-700 group-focus-within:text-[var(--color)] group-hover:group-focus-within:text-[var(--color)] cursor-pointer',
                error ? 'text-red-500/80' : 'text-black/50',
                `placement-${labelPlacement}`,
                (labelPlacement === 'inset' || hasInputValue) && 'shrink',
                theme.label,
                classes.label
              )}
              for={id}
              bind:this={labelEl}
            >
              {label}
            </label>
          {/if}

          <div
            class={cls(
              'input col-span-full row-span-full flex items-center',
              hasInsetLabel && 'pt-4',
              dense ? 'my-1' : 'my-2',
              (hasPrefix || hasSuffix) &&
                label &&
                labelPlacement === 'float' &&
                !hasInputValue &&
                'opacity-0 transition-opacity',
              'group-focus-within:opacity-100'
            )}
          >
            <slot name="prefix" />

            {#if type === 'currency'}
              <Icon path={mdiCurrencyUsd} size="1.1em" class="text-black/50 -mt-1" />
            {/if}

            {#if multiline}
              <textarea
                {id}
                {name}
                {placeholder}
                {autocomplete}
                value={inputValue}
                {autocapitalize}
                on:input={handleInput}
                on:focus
                on:blur
                on:keydown
                on:keypress
                class={cls(
                  'text-sm border-none w-full bg-transparent outline-none resize-none',
                  'placeholder-black placeholder-opacity-0 group-focus-within:placeholder-opacity-30',
                  error && 'placeholder-red-800',
                  (labelPlacement !== 'float' || !hasInsetLabel) && 'placeholder-opacity-30',
                  {
                    'text-left': align === 'left',
                    'text-center': align === 'center',
                    'text-right': align === 'right',
                  },
                  theme.input,
                  classes.input
                )}
                use:multi={actions}
              />
            {:else}
              <Input
                {id}
                {name}
                {placeholder}
                {autocomplete}
                type={inputType}
                inputmode={inputMode}
                value={inputValue}
                {mask}
                {replace}
                {accept}
                {autocapitalize}
                {actions}
                bind:inputEl
                on:input={handleInput}
                on:focus
                on:blur
                on:keydown
                on:keypress
                class={cls(
                  'text-sm border-none w-full bg-transparent outline-none truncate',
                  'selection:bg-gray-500/30',
                  'placeholder-black placeholder-opacity-0 group-focus-within:placeholder-opacity-30',
                  error && 'placeholder-red-800',
                  (labelPlacement !== 'float' || !hasInsetLabel) && 'placeholder-opacity-30',
                  {
                    'text-left': align === 'left',
                    'text-center': align === 'center',
                    'text-right': align === 'right',
                  },
                  theme.input,
                  classes.input
                )}
              />
            {/if}

            {#if type === 'percent'}
              <Icon path={mdiPercent} size="1.1em" class="text-black/50 -mt-1 ml-1" />
            {/if}

            <slot name="suffix" />
          </div>
        </div>

        {#if hasAppend}
          <div class={cls('append whitespace-nowrap', theme.append, classes.append)}>
            {#if clearable && hasInputValue}
              <Button
                icon={mdiClose}
                class="text-black/50 p-1"
                on:click={() => {
                  inputValue = '';
                  operator = operators ? operators[0].value : null;
                  updateValue();
                  dispatch('clear');
                  labelEl?.focus();
                }}
              />
            {/if}

            {#if operators}
              <select
                value={operator}
                on:change={(e) => {
                  operator = e.target.value;
                  updateValue();
                }}
                class="appearance-none bg-black/5 border border-black/20 rounded-full mr-2 px-2 text-sm outline-none focus:border-opacity-50 focus:shadow-md"
                style="text-align-last: center;"
              >
                {#each operators ?? [] as { label, value }}
                  <option {value}>{label}</option>
                {/each}
              </select>
            {/if}

            {#if type === 'password'}
              <Button
                icon={mdiEye}
                class="text-black/50 p-2"
                on:click={() => {
                  if (inputType === 'password') {
                    inputType = 'text';
                  } else {
                    inputType = 'password';
                  }
                }}
              />
            {/if}

            <slot name="append" />

            {#if error}
              <Icon path={mdiInformationOutline} class="text-red-500" />
            {:else if iconRight}
              <Icon path={iconRight} class="text-black/50" />
            {/if}
          </div>
        {/if}
      </div>
    </div>

    <div
      class={cls(
        error ? 'error' : 'hint',
        'text-xs ml-2 transition-transform ease-out overflow-hidden origin-top transform group-focus-within:scale-y-100',
        error ? 'text-red-500' : 'text-black/50 scale-y-0',
        theme.error,
        classes.error
      )}
    >
      {error && error != true ? error : hint}
    </div>
  </div>
</fieldset>

<style lang="postcss">
  fieldset:focus-within label.placement-float,
  label.shrink {
    transform: scale(0.75);
    width: 133%; /* offset 75% scale */
    height: 32px;
  }

  :global(input::placeholder),
  textarea::placeholder {
    transition: color 200ms; /* Move to tailwind plugin? */
  }
</style>
