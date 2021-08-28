<template>
  <q-layout view="lHh Lpr lFf">
    <q-header elevated>
      <q-toolbar>
        <q-btn
          flat
          dense
          round
          icon="menu"
          aria-label="Menu"
          @click="toggleLeftDrawer"
        />

        <q-toolbar-title> Friendly chat </q-toolbar-title>

        <div v-if="user">
          <q-avatar><q-img :src="user.photoURL" /></q-avatar>
          {{ user.displayName }}
        </div>
        <q-btn v-else-if="user === null" @click="signIn">
          Sign in with Google
        </q-btn>
        <q-spinner v-else />
      </q-toolbar>
    </q-header>

    <q-drawer v-model="leftDrawerOpen" show-if-above bordered class="bg-grey-1">
      <q-list>
        <q-item-label header class="text-grey-8">
          Rooms
          <q-spinner v-if="!data" />
        </q-item-label>
        <q-item v-for="(room, name) in data?.rooms" :key="name">
          <q-btn
            :color="currentRoom === name ? 'primary' : ''"
            flat
            @click="showRoom(name)"
            >#{{ name }}</q-btn
          >
        </q-item>
        <q-item><q-btn @click="createRoom">Create room</q-btn></q-item>
      </q-list>
    </q-drawer>

    <q-page-container>
      <q-list v-if="currentRoom">
        <q-item
          v-for="(msg, id) in data?.rooms[currentRoom]?.messages"
          :key="id"
        >
          <q-item-section avatar>
            <q-avatar>
              <q-img :src="msg.user.avatar" />
            </q-avatar>
          </q-item-section>
          <q-item-section>
            <q-item-label>{{ msg.user.name }}</q-item-label>
            <q-item-label caption> {{ msg.message }} </q-item-label>
          </q-item-section>
        </q-item>
        <q-item>
          <q-item-section>
            <q-form @submit="createMessage(message)">
              <q-input
                outlined
                v-model="message"
                label="Say something"
                counter
                maxlength="200"
              >
                <template v-slot:before>
                  <q-avatar>
                    <img :src="user.photoURL" />
                  </q-avatar>
                </template>

                <template v-slot:after>
                  <q-btn
                    round
                    dense
                    flat
                    icon="send"
                    type="submit"
                    @click="createMessage(message)"
                  />
                </template>
              </q-input>
            </q-form>
          </q-item-section>
        </q-item>
      </q-list>
      <!-- <router-view /> -->
    </q-page-container>
  </q-layout>
</template>

<script lang="ts">
import {
  getAuth,
  signInWithPopup,
  GoogleAuthProvider,
  onAuthStateChanged,
  User,
} from 'firebase/auth';
import { useQuasar } from 'quasar';
import { defineComponent, ref, shallowRef, watchEffect, computed } from 'vue';
import {
  getDatabase,
  ref as rtdbRef,
  child,
  onValue,
  off,
  push,
  DatabaseReference,
} from 'firebase/database';

type FirebaseDatabaseResult<T> =
  | Readonly<{ path: string; data?: T; dbRef: DatabaseReference }>
  | undefined;

function useRTDB<T>(fn: () => string) {
  const result = shallowRef<FirebaseDatabaseResult<T>>();
  watchEffect((onInvalidate) => {
    const path = fn();
    const dbRef = path && rtdbRef(getDatabase(), path);
    dbRef &&
      onValue(dbRef, (snapshot) => {
        result.value = {
          path,
          dbRef,
          data: snapshot.val() as T,
        };
      });
    onInvalidate(() => {
      dbRef && off(dbRef);
    });
  });
  return result;
}
type FirebaseUserResult = Readonly<User> | null | undefined;

function useFirebaseUser() {
  const result = shallowRef<FirebaseUserResult>();
  onAuthStateChanged(getAuth(), (user) => {
    result.value = user;
  });
  return result;
}

interface Chat {
  rooms?: {
    [name: string]: Room;
  };
}

interface Room {
  messages: {
    [id: string]: Message;
  };
}

interface Message {
  user: {
    name: string;
    avatar: string;
  };
  text: string;
}

export default defineComponent({
  name: 'MainLayout',

  components: {},

  setup() {
    const leftDrawerOpen = ref(false);
    const currentRoom = ref<string>();
    const $q = useQuasar();
    const message = ref('');

    const provider = new GoogleAuthProvider();
    const signIn = async () => await signInWithPopup(getAuth(), provider);

    const user = useFirebaseUser();
    const rtdb = useRTDB<Chat>(() => user && '/');

    async function createMessage(message: string) {
      if (!user.value) {
        throw new Error('Unathenticated user!');
      }
      if (!rtdb.value) {
        throw new Error('Database not ready!');
      }
      if (!currentRoom.value) {
        throw new Error('No chat room selected!');
      }
      await push(
        child(rtdb.value.dbRef, `rooms/${currentRoom.value}/messages`),
        {
          user: { name: user.value.displayName, avatar: user.value.photoURL },
          message,
        }
      );
    }

    return {
      data: computed(() => rtdb.value?.data),
      signIn,
      createMessage,
      message,
      user,
      leftDrawerOpen,
      currentRoom,
      createRoom() {
        $q.dialog({
          title: 'Create room',
          message: 'Room name?',
          prompt: {
            model: '',
            type: 'text', // optional
          },
          cancel: true,
          persistent: true,
        }).onOk(async (name: string) => {
          currentRoom.value = name;
          await createMessage('Created new room!');
        });
      },
      showRoom(name: string) {
        currentRoom.value = name;
      },
      toggleLeftDrawer() {
        leftDrawerOpen.value = !leftDrawerOpen.value;
      },
    };
  },
});
</script>
