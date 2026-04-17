<script setup lang="ts">
import { RouterLink, RouterView } from 'vue-router'
import HelloWorld from './components/HelloWorld.vue'
import axios from 'axios'
import { ref } from 'vue'

const advice = ref('')
const isloading = ref(false)

const fetchAdvice = async () => {
  isloading.value = true
  try {
    const response = await axios.get('https://api.adviceslip.com/advice')
    advice.value = response.data.slip.advice
  } catch (err) {
    advice.value = 'Can not connect network, please check installed axios'
  } finally {
    isloading.value = false
  }
}
</script>

<template>
  <header>
    <img alt="Vue logo" class="logo" src="@/assets/logo.svg" width="125" height="125" />

    <div class="wrapper">
      <HelloWorld msg="You did it!" />
      <section>
        <div class="advice-box">
          <p v-if="!isloading">{{ advice || 'Click button to fetch advice' }}</p>
          <p v-else class="loading">Connecting Advice API ...</p>
        </div>
        <button @click="fetchAdvice" :disabled="isloading">
          Fetch Advice
        </button>
      </section>


      <nav>
        <RouterLink to="/">Home</RouterLink>
        <RouterLink to="/about">About</RouterLink>
      </nav>
    </div>
  </header>

  <RouterView />
</template>

<style scoped>
header {
  line-height: 1.5;
  max-height: 100vh;
}

.logo {
  display: block;
  margin: 0 auto 2rem;
}

nav {
  width: 100%;
  font-size: 12px;
  text-align: center;
  margin-top: 2rem;
}

nav a.router-link-exact-active {
  color: var(--color-text);
}

nav a.router-link-exact-active:hover {
  background-color: transparent;
}

nav a {
  display: inline-block;
  padding: 0 1rem;
  border-left: 1px solid var(--color-border);
}

nav a:first-of-type {
  border: 0;
}

@media (min-width: 1024px) {
  header {
    display: flex;
    place-items: center;
    padding-right: calc(var(--section-gap) / 2);
  }

  .logo {
    margin: 0 2rem 0 0;
  }

  header .wrapper {
    display: flex;
    place-items: flex-start;
    flex-wrap: wrap;
  }

  nav {
    text-align: left;
    margin-left: -1rem;
    font-size: 1rem;

    padding: 1rem 0;
    margin-top: 1rem;
  }
}
</style>
