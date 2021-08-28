<template>
  <router-view />
</template>
<script lang="ts">
import { defineComponent } from 'vue';
import { getApps, initializeApp, FirebaseOptions } from 'firebase/app';

// Your web app's Firebase configuration
const firebaseConfig: FirebaseOptions = {
  apiKey: 'AIzaSyChod4nW3zTHJjQ5maubsDyz3M6kN9ZhDQ',
  authDomain: 'friendly-chat-f6424.firebaseapp.com',
  projectId: 'friendly-chat-f6424',
  storageBucket: 'friendly-chat-f6424.appspot.com',
  messagingSenderId: '923282218906',
  appId: '1:923282218906:web:b2c2905bae0752dc21a423',
};

export default defineComponent({
  name: 'App',

  setup() {
    if (!getApps().length) {
      initializeApp(firebaseConfig);
    }
  },
});
</script>
