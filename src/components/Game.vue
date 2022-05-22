<script setup lang="ts">
import Fraction from "fraction.js";
import {fractionToMonzo, getRank2Name, Temperament} from "temperaments";

defineProps<{
  msg: string
}>()

const syntonic = new Fraction(81, 80);
const comma = fractionToMonzo(syntonic);
const temperament = Temperament.fromCommaList([comma]);
temperament.canonize();
const meantone = getRank2Name(temperament.subgroup, temperament.rank2Prefix());

</script>

<template>
  <div class="greetings">
    <h1 class="green">{{ msg }}</h1>
    <h3>
      Herpty derp play in {{ meantone }}
    </h3>
  </div>
</template>

<style scoped>
h1 {
  font-weight: 500;
  font-size: 2.6rem;
  top: -10px;
}

h3 {
  font-size: 1.2rem;
}

.greetings h1,
.greetings h3 {
  text-align: center;
}

@media (min-width: 1024px) {
  .greetings h1,
  .greetings h3 {
    text-align: left;
  }
}
</style>
