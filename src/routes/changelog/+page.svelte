<script lang="ts">
  import { marked } from 'marked';

  import changelog from '../../../CHANGELOG.md?raw';
  import TableOfContents from '$lib/components/TableOfContents.svelte';

  function sanitize(str: string) {
    return str.replace(/</g, '\\<').replace(/>/g, '\\>');
  }
</script>

<div class="grid grid-cols-[1fr,auto] gap-6 pt-2 pb-4">
  <div class="bg-white p-2 m-2 rounded shadow-lg border overflow-auto">
    <div class="prose px-4">
      {@html marked.parse(sanitize(changelog))}
    </div>
  </div>

  <div class="hidden lg:block w-[224px]">
    <div class="sticky top-0 pr-2 max-h-[calc(100vh-64px)] overflow-auto">
      <div class="text-xs uppercase leading-8 tracking-widest text-black/50">On this page</div>
      <TableOfContents maxDepth={2} />
    </div>
  </div>
</div>
