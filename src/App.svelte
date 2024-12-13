<script lang="ts">
  import { v4 as uuidv4 } from 'uuid';
  import KeyValue from './lib/KeyValue.svelte';
  import type { Account, User } from "./lib/types";
  import Users from "./lib/Users.svelte";
  import Value from "./lib/Value.svelte";
  import Accounts from "./lib/Accounts.svelte";

  let db = $state<IDBDatabase | null>(null);

  const open = async (databaseName: string, newVersion?: { version?: number, upgrade: (db: IDBDatabase) => Promise<void> }): Promise<IDBDatabase> => {
    return new Promise((resolve, reject) => {
      const openDBRequest = window.indexedDB.open(databaseName, newVersion ? newVersion.version : undefined);
      openDBRequest.onsuccess = (_: Event) => {
        console.log('open - success');
        resolve(openDBRequest.result);
      };
      openDBRequest.onerror = (event: Event) => {
        console.log('open - error');
        console.error(event);
        reject('open failed');
      };
      if (newVersion) {
        openDBRequest.onupgradeneeded = async (_: IDBVersionChangeEvent) => {
          try {
            console.log('open - upgrade started');
            const dbResult = openDBRequest.result;
            await newVersion.upgrade(dbResult);
            db = dbResult;
            console.log('open - upgrade finished');
          } catch (error) {
            openDBRequest.transaction?.abort();
            console.log('open - upgrade aborted');
          }
        };
        openDBRequest.onblocked = (event: IDBVersionChangeEvent) => {
          console.log('open - blocked');
          console.error(event);
          alert('blocked');
          reject('blocked');
        };
      }
    });
  };

  const close = () => {
    if (db) {
      db.close();
    }
  };

  const addEventListeners = (db: IDBDatabase) => {
    db.onclose = (_: Event) => {
      console.error('db close');
    };
    db.onversionchange = (_: Event) => {
      console.error('db version change');
      alert('db version change');
    };
  };

  const createDefault = async () => {
    const openDBRequest = window.indexedDB.open('eg');
    openDBRequest.onsuccess = (_: Event) => {
      console.log('createDefault - success');
      db = openDBRequest.result;
      addEventListeners(db);
      //
      // const t = db!.transaction(['users'], 'readwrite');
      //
      // t.onabort = () => console.log('abort');
      // t.onerror = () => console.log('error');
      // t.oncomplete = () => console.log('complete');
      //
      // const s = t.objectStore('users');
      // const r = s.add({});
      // r.onerror = () => console.log('error');
      // r.onsuccess = () => console.log('success');
      // t.commit();
    };
    openDBRequest.onerror = (event: Event) => {
      console.log('createDefault - error');
      console.error(event);
    };
    openDBRequest.onupgradeneeded = (_: IDBVersionChangeEvent) => {
      console.log('createDefault - upgrade started');
      const dbResult = openDBRequest.result;

      dbResult.createObjectStore('values', { autoIncrement: true });
      dbResult.createObjectStore('key_values');
      dbResult.createObjectStore('accounts', { autoIncrement: true, keyPath: 'id' });
      const usersStore = dbResult.createObjectStore('users', { keyPath: 'id' });
      usersStore.createIndex('email_index', 'email', { unique: true });

      usersStore.transaction.oncomplete = (_) => {
        const transaction = dbResult.transaction(['values', 'key_values', 'accounts', 'users'], 'readwrite', { durability: 'strict' });

        const values = transaction.objectStore('values');
        values.add('value_1');
        values.add('value_2');

        const keyValues = transaction.objectStore('key_values');
        keyValues.add('value_1', 'key_1');
        keyValues.add('value_2', 'key_2');

        const accounts = transaction.objectStore('accounts');
        const account1: Account = {
          name: 'account1',
          netWorth: 123_000,
        };
        accounts.add(account1);
        const account2: Account = {
          name: 'account2',
          netWorth: 234_000,
        };
        accounts.add(account2);

        const users = transaction.objectStore('users');
        const user1: User = {
          email: 'p1@email.com',
          id: uuidv4(),
          name: 'p1',
        };
        users.add(user1);
      };
      usersStore.transaction.onerror = (event: Event) => {
        console.log('createDefault - usersStore error');
        console.error(event);
      };
      usersStore.transaction.onabort = (event: Event) => {
        console.log('createDefault - usersStore abort');
        console.error(event);
      };

      console.log('createDefault - upgrade finished');
    };

    openDBRequest.onblocked = (event: IDBVersionChangeEvent) => {
      console.log('createDefault - blocked');
      console.error(event);
    };
  };

  const upgrade = async () => {
    close();
    db = await open(
      'eg',
      {
        version: db!.version + 1,
        upgrade: async (db) => {
          console.log('upgrade - started');
          const newStore = db.createObjectStore('new');
          // db.createObjectStore('new2');
          // console.log('upgrade - 1');
          // const r = db.transaction('new2', 'readwrite').objectStore('new2').put('qqq');
          // console.log('upgrade - 2');
          // r.onerror = () => {
          //   console.error(r.error);
          // };
          // console.log('upgrade - 3');

          newStore.transaction.oncomplete = (_) => {
            console.log('upgrade - transaction completed');
          };
          newStore.transaction.onerror = (event: Event) => {
            console.log('upgrade - transaction error');
            console.error(event);
          };
          newStore.transaction.onabort = (event: Event) => {
            console.log('upgrade - transaction abort');
            console.error(event);
          };
          console.log('upgrade - finished');
        },
      },
    );
    addEventListeners(db);
  };

  const reset = async () => {
    close();
    console.log('reset - delete started');
    const openDBRequest = window.indexedDB.deleteDatabase('eg');
    openDBRequest.onsuccess = async (_: Event) => {
      console.log('reset - delete success');
      db = null;
      await createDefault();
    };
    openDBRequest.onerror = (event: Event) => {
      console.log('reset - delete error');
      console.error(event);
    };
    openDBRequest.onblocked = (event: Event) => {
      console.log('reset - delete blocked');
      console.error(event);
      console.log(openDBRequest.transaction);
    };
    console.log('reset - delete finished');
  };

  createDefault();
</script>

<main>
  <div class="flex flex-col gap-4">
    {#if db}
      <span>{db.name}&nbsp;(version: {db.version}) - <button onclick="{() => reset()}" type="button">Reset</button>&nbsp;<button onclick="{() => upgrade()}" type="button">Upgrade</button>&nbsp;<button onclick="{() => close()}" type="button">Close</button></span>
      <hr class="w-full" />
      <div class="flex flex-col gap-4 items-center">
        <Value db={db} />
        <hr class="w-full" />
        <KeyValue db={db} />
        <hr class="w-full" />
        <Accounts db={db} />
        <hr class="w-full" />
        <Users db={db} />
      </div>
    {/if}
  </div>
</main>

<style>
</style>
