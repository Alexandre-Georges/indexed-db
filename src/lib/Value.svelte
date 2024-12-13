<script lang="ts">
  import type { KeyValue } from "./types";

  interface Props {
    db: IDBDatabase;
  }
  let props: Props = $props();
  let count = $state(0);
  let keyValues = $state<KeyValue[]>([]);

  const refresh = async () => {
    return new Promise((resolve, reject) => {
      const valuesStore = props.db.transaction('values', 'readonly').objectStore('values');

      const request = valuesStore.openCursor();
      const tmpKeyValues: KeyValue[] = [];
      request.onsuccess = (_: Event) => {
        let cursor = request.result;
        if (cursor) {
          tmpKeyValues.push({ key: cursor.key as string, value: cursor.value });
          cursor.continue();
        } else {
          keyValues = tmpKeyValues;
          resolve('');
        }
      };
      request.onerror = (event: Event) => {
        console.error(event);
        reject('request failed');
      };

      const countRequest = valuesStore.count();
      countRequest.onsuccess = (_: Event) => {
        count = countRequest.result;
        resolve('');
      };
      countRequest.onerror = (event: Event) => {
        console.error(event);
        reject('request failed');
      };
    });
  };

  refresh();

  const onClear = () => {
    const valuesStore = props.db.transaction('values', 'readwrite').objectStore('values');
    const request = valuesStore.clear();
    request.onsuccess = async (_: Event) => {
      await refresh();
    };
    request.onerror = async (event: Event) => {
      console.error(event);
    };
  };

  let value = $state('');

  const onSubmit = () => {
    if (value !== '') {
      const transaction = props.db.transaction('values', 'readwrite');
      try {
        const valuesStore = transaction.objectStore('values');
        const request = valuesStore.add(value);

        request.onsuccess = async (_: Event) => {
          await refresh();
          value = '';
        };
        request.onerror = async (event: Event) => {
          console.error(event);
        };
        transaction.commit();
      } catch (error) {
        transaction.abort();
      }
    }
  };

  const onDelete = async (key: string) => {
    const valuesStore = props.db.transaction('values', 'readwrite').objectStore('values');
    const request = valuesStore.delete(key);
    request.onsuccess = async (_: Event) => {
      await refresh();
    };
    request.onerror = async (event: Event) => {
      console.error(event);
    };
  };
</script>

<div class="flex flex-col gap-4 items-center">
  <span>Values</span>
  <div>{count} element(s) - <button onclick={onClear} type="button">Clear</button></div>
  <form onsubmit={event => {
    event.preventDefault();
    onSubmit();
  }} class="flex flex-col w-2/4 max-w-40">
    <label for="value">Value</label>
    <input bind:value={value} id="value" type="text" />
    <button type="submit">Save</button>
  </form>
  <div class="grid grid-cols-3 border-b w-2/4 min-w-96">
    <span class="px-1 py-0.5 border-t border-l font-bold">Key</span>
    <span class="px-1 py-0.5 border-t border-l font-bold">Value</span>
    <span class="px-1 py-0.5 border-t border-l border-r font-bold"></span>
    {#each keyValues as keyValue}
      <span class="px-1 py-0.5 border-t border-l">{keyValue.key}</span>
      <span class="px-1 py-0.5 border-t border-l">{keyValue.value}</span>
      <span class="px-1 py-0.5 border-t border-l border-r text-center">
        <button onclick={() => onDelete(keyValue.key)} type="button">Delete</button>
      </span>
    {/each}
  </div>
</div>
