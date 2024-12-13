<script lang="ts">
  import { v4 as uuidv4 } from 'uuid';
  import type { User } from "./types";

  interface Props {
    db: IDBDatabase;
  }
  let props: Props = $props();
  let count = $state(0);
  let users = $state<User[]>([]);

  const refresh = async () => {
    return new Promise((resolve, reject) => {
      const usersStore = props.db.transaction('users', 'readonly').objectStore('users');

      const request = usersStore.openCursor();
      const tmpUsers: User[] = [];
      request.onsuccess = (_: Event) => {
        let cursor = request.result;
        if (cursor) {
          tmpUsers.push(cursor.value);
          cursor.continue();
        } else {
          users = tmpUsers;
          resolve('');
        }
      };
      request.onerror = (event: Event) => {
        console.error(event);
        reject('request failed');
      };

      const countRequest = usersStore.count();
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
    const usersStore = props.db.transaction('users', 'readwrite').objectStore('users');
    const request = usersStore.clear();
    request.onsuccess = async (_: Event) => {
      await refresh();
    };
    request.onerror = async (event: Event) => {
      console.error(event);
    };
  };

  const getNewUser = () => {
    return `{
  "email": "p2@email.com",
  "id": "${uuidv4()}",
  "name": "p2"
}`;
  };

  let newUser = $state(getNewUser());

  const onSubmit = () => {
    if (newUser !== '') {
      const transaction = props.db.transaction('users', 'readwrite');
      try {
      const usersStore = transaction.objectStore('users');
        const request = usersStore.add(JSON.parse(newUser));
        request.onsuccess = async (_: Event) => {
          await refresh();
          newUser = getNewUser();
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
    const usersStore = props.db.transaction('users', 'readwrite').objectStore('users');
    const request = usersStore.delete(key);
    request.onsuccess = async (_: Event) => {
      await refresh();
    };
    request.onerror = async (event: Event) => {
      console.error(event);
    };
  };
</script>

<div class="flex flex-col gap-4 items-center">
  <span>Users</span>
  <div>{count} element(s) - <button onclick={onClear} type="button">Clear</button></div>
  <form onsubmit={event => {
    event.preventDefault();
    onSubmit();
  }} class="flex flex-col w-2/4 max-w-2xl">
    <label for="new-user">New user</label>
    <textarea bind:value={newUser} id="new-user" rows="10"></textarea>
    <button type="submit">Save</button>
  </form>
  <div class="grid grid-cols-5 border-b min-w-96">
    <span class="col-span-2 px-1 py-0.5 border-t border-l font-bold">Key</span>
    <span class="col-span-2 px-1 py-0.5 border-t border-l font-bold">Value</span>
    <span class="col-span-1 px-1 py-0.5 border-t border-l border-r font-bold"></span>
    {#each users as user}
      <span class="col-span-2 px-1 py-0.5 border-t border-l">{user.id}</span>
      <pre class="col-span-2 px-1 py-0.5 border-t border-l">{JSON.stringify(user, undefined, 2)}</pre>
      <span class="col-span-1 px-1 py-0.5 border-t border-l border-r text-center">
        <button onclick={() => onDelete(user.id)} type="button">Delete</button>
      </span>
    {/each}
  </div>
</div>
