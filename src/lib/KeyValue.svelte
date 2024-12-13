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
      const keyValuesStore = props.db.transaction('key_values', 'readonly').objectStore('key_values');

      const request = keyValuesStore.openCursor();
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

      const countRequest = keyValuesStore.count();
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
    const keyValuesStore = props.db.transaction('key_values', 'readwrite').objectStore('key_values');
    const request = keyValuesStore.clear();
    request.onsuccess = async (_: Event) => {
      await refresh();
    };
    request.onerror = async (event: Event) => {
      console.error(event);
    };
  };

  let key = $state('');
  let value = $state('');

  const onSubmit = () => {
    if (key !== '' && value !== '') {
      const transaction = props.db.transaction('key_values', 'readwrite');
      try {
        const keyValuesStore = transaction.objectStore('key_values');
        const request = keyValuesStore.add(value, key);
        request.onsuccess = async (_: Event) => {
          await refresh();
          key = '';
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
    const keyValuesStore = props.db.transaction('key_values', 'readwrite').objectStore('key_values');
    const request = keyValuesStore.delete(key);
    request.onsuccess = async (_: Event) => {
      await refresh();
    };
    request.onerror = async (event: Event) => {
      console.error(event);
    };
  };
</script>

<div class="flex flex-col gap-4 items-center">
  <span>Key values</span>
  <div>{count} element(s) - <button onclick={onClear} type="button">Clear</button></div>
  <form onsubmit={event => {
    event.preventDefault();
    onSubmit();
  }} class="flex flex-col w-2/4 max-w-40">
    <label for="key">Key</label>
    <input bind:value={key} id="key" type="text" />
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
