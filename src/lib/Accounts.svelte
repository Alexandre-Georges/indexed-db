<script lang="ts">
  import type { Account } from "./types";

  interface Props {
    db: IDBDatabase;
  }
  let props: Props = $props();
  let count = $state(0);
  let accounts = $state<Account[]>([]);

  const refresh = async () => {
    return new Promise((resolve, reject) => {
      const accountsStore = props.db.transaction('accounts', 'readonly').objectStore('accounts');

      const request = accountsStore.openCursor();
      const tmpAccounts: Account[] = [];
      request.onsuccess = (_: Event) => {
        let cursor = request.result;
        if (cursor) {
          tmpAccounts.push(cursor.value);
          cursor.continue();
        } else {
          accounts = tmpAccounts;
          resolve('');
        }
      };
      request.onerror = (event: Event) => {
        console.error(event);
        reject('request failed');
      };

      const countRequest = accountsStore.count();
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
    const accountsStore = props.db.transaction('accounts', 'readwrite').objectStore('accounts');
    const request = accountsStore.clear();
    request.onsuccess = async (_: Event) => {
      await refresh();
    };
    request.onerror = async (event: Event) => {
      console.error(event);
    };
  };

  const getNewAccount = () => {
    return `{
  "name": "Account name",
  "netWorth": 100000
}`;
  };

  let newAccount = $state(getNewAccount());

  const onSubmit = () => {
    if (newAccount !== '') {
      const transaction = props.db.transaction('accounts', 'readwrite')
      try {
        const accountsStore = transaction.objectStore('accounts');
        const request = accountsStore.add(JSON.parse(newAccount));
        request.onsuccess = async (_: Event) => {
          await refresh();
          newAccount = getNewAccount();
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

  const onDelete = async (key: string | undefined) => {
    const accountsStore = props.db.transaction('accounts', 'readwrite').objectStore('accounts');
    const request = accountsStore.delete(key as string);
    request.onsuccess = async (_: Event) => {
      await refresh();
    };
    request.onerror = async (event: Event) => {
      console.error(event);
    };
  };
</script>

<div class="flex flex-col gap-4 items-center">
  <span>Accounts</span>
  <div>{count} element(s) - <button onclick={onClear} type="button">Clear</button></div>
  <form onsubmit={event => {
    event.preventDefault();
    onSubmit();
  }} class="flex flex-col w-2/4 max-w-2xl">
    <label for="new-account">New account</label>
    <textarea bind:value={newAccount} id="new-account" rows="10"></textarea>
    <button type="submit">Save</button>
  </form>
  <div class="grid grid-cols-5 border-b min-w-96">
    <span class="col-span-2 px-1 py-0.5 border-t border-l font-bold">Key</span>
    <span class="col-span-2 px-1 py-0.5 border-t border-l font-bold">Value</span>
    <span class="col-span-1 px-1 py-0.5 border-t border-l border-r font-bold"></span>
    {#each accounts as account}
      <span class="col-span-2 px-1 py-0.5 border-t border-l">{account.id}</span>
      <pre class="col-span-2 px-1 py-0.5 border-t border-l">{JSON.stringify(account, undefined, 2)}</pre>
      <span class="col-span-1 px-1 py-0.5 border-t border-l border-r text-center">
        <button onclick={() => onDelete(account.id)} type="button">Delete</button>
      </span>
    {/each}
  </div>
</div>
