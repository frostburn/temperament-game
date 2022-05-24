<script setup lang="ts">
import { RouterLink, RouterView } from "vue-router";
import HelloWorld from "@/components/HelloWorld.vue";
import { reactive, ref } from "vue";

const subgroup = reactive([0, 1, 2]);
const showWarts = ref(false);

function selectSubgroup(event: Event) {
  while (subgroup.length) {
    subgroup.pop();
  }
  const value = (event.target as HTMLSelectElement)!.value;
  if (value === "2.3.7") {
    subgroup.push(0);
    subgroup.push(1);
    subgroup.push(3);
  } else if (value === "2.5.7") {
    subgroup.push(0);
    subgroup.push(2);
    subgroup.push(3);
  } else {
    subgroup.push(0);
    subgroup.push(1);
    subgroup.push(2);
  }
}
</script>

<template>
  <header>
    <img
      alt="Vue logo"
      class="logo"
      src="@/assets/logo.svg"
      width="125"
      height="125"
    />

    <div class="wrapper">
      <HelloWorld msg='Projective Tuning Space "Game"' />
      <div>
        <label for="show-warts">Show warts </label>
        <input id="show-warts" type="checkbox" v-model="showWarts" />

        <label for="subgroup"> Subgroup </label>
        <select id="subgroup" @change="selectSubgroup">
          <option>5-limit</option>
          <option>2.3.7</option>
          <option>2.5.7</option>
        </select>
      </div>

      <nav>
        <RouterLink to="/">Home</RouterLink>
        <RouterLink to="/about">About</RouterLink>
      </nav>
    </div>
  </header>

  <RouterView :subgroup="subgroup" :showWarts="showWarts" />
</template>

<style>
@import "@/assets/base.css";

#app {
  max-width: 1280px;
  margin: 0 auto;
  padding: 2rem;

  font-weight: normal;
}

header {
  line-height: 1.5;
  max-height: 100vh;
}

.logo {
  display: block;
  margin: 0 auto 2rem;
}

a,
.green {
  text-decoration: none;
  color: hsla(160, 100%, 37%, 1);
  transition: 0.4s;
}

@media (hover: hover) {
  a:hover {
    background-color: hsla(160, 100%, 37%, 0.2);
  }
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
  body {
    display: flex;
    place-items: center;
  }

  #app {
    padding: 0 2rem;
  }

  header {
    display: flex;
    place-items: center;
    padding-right: calc(var(--section-gap) / 2);
  }

  .logo {
    margin: 0 2rem 0 0;
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
