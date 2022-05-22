<script setup lang="ts">
import Algebra from "ganja.js";
import { gcd, inverseLogMetric, patentVal } from "temperaments";
import { computed, reactive, ref } from "vue";

defineProps<{
  subgroup: string;
}>();

const Cl3 = Algebra(3);

const edo9 = patentVal(9, 2, 3);
const edo15 = patentVal(15, 2, 3);
const edo12 = patentVal(12, 2, 3);
const edo19 = patentVal(19, 2, 3);
const edo22 = patentVal(22, 2, 3);

const vals = reactive([edo12, edo19, edo22, edo9, edo15]);
const selected = ref(-1);

const valPairs = reactive([
  [0, 1],
  [0, 2],
  [1, 2],
]);

const diagonalYScale = Math.sqrt(0.75);

const points = computed(() => {
  const metric = inverseLogMetric([0, 1, 2]);
  return vals.map((val, index) => {
    let a = val[0] * metric[0];
    let b = val[1] * metric[1];
    let c = val[2] * metric[2];
    const normalizer = 1 / Math.sqrt(a * a + b * b + c * c);

    a *= normalizer;
    b *= normalizer;
    c *= normalizer;
    const x = 0.5 * (a + b) - c;
    const y = (a - b) * diagonalYScale;
    return {
      x: x * 20000 + 200,
      y: y * 20000 + 200,
      fill: index === selected.value ? "green" : "black",
    };
  });
});

const lines = computed(() => {
  const ps = points.value;
  const main = valPairs.map((pair) => {
    return {
      x1: ps[pair[0]].x,
      y1: ps[pair[0]].y,
      x2: ps[pair[1]].x,
      y2: ps[pair[1]].y,
      strokeDasharray: "",
    };
  });
  const bonus0 = valPairs.map((pair) => {
    return {
      x1: ps[pair[0]].x,
      y1: ps[pair[0]].y,
      x2: 2 * ps[pair[0]].x - ps[pair[1]].x,
      y2: 2 * ps[pair[0]].y - ps[pair[1]].y,
      strokeDasharray: "5,5",
    };
  });
  const bonus1 = valPairs.map((pair) => {
    return {
      x1: 2 * ps[pair[1]].x - ps[pair[0]].x,
      y1: 2 * ps[pair[1]].y - ps[pair[0]].y,
      x2: ps[pair[1]].x,
      y2: ps[pair[1]].y,
      strokeDasharray: "5,5",
    };
  });
  return main.concat(bonus0).concat(bonus1);
});

function select(index: number) {
  if (selected.value < 0) {
    selected.value = index;
  } else if (selected.value !== index) {
    const pair = [selected.value, index];
    let novel = true;
    valPairs.forEach((p) => {
      if (p[0] === pair[0] && p[1] === pair[1]) {
        novel = false;
      }
      if (p[0] === pair[1] && p[1] === pair[0]) {
        novel = false;
      }
    });
    if (novel) {
      valPairs.push(pair);
      checkIntersections();
      selected.value = -1;
    }
  }
}

function checkIntersections() {
  const vectors = vals.map((val) => {
    const vector = Array(8).fill(0);
    vector.splice(1, 3, ...val);
    return new Cl3(vector);
  });

  const wedgies = valPairs.map((pair) =>
    vectors[pair[0]].Wedge(vectors[pair[1]])
  );
  for (let i = 0; i < wedgies.length; ++i) {
    for (let j = 0; j < wedgies.length; ++j) {
      const intersection = wedgies[i].Vee(wedgies[j]);
      if (intersection[1] < 0) {
        for (let k = 0; k < intersection.length; ++k) {
          intersection[k] = -intersection[k];
        }
      }
      const val = [...intersection.slice(1, 4)];
      if (!val[0] && !val[1] && !val[2]) {
        continue;
      }
      const commonFactor = val.reduce(gcd);
      for (let k = 0; k < val.length; ++k) {
        val[k] = Math.round(val[k] / commonFactor);
      }
      let novel = true;
      vals.forEach((v) => {
        if (v[0] === val[0] && v[1] === val[1] && v[2] === val[2]) {
          novel = false;
        }
      });
      if (novel) {
        vals.push(val);
      }
    }
  }
}
</script>

<template>
  <div class="greetings">
    <h1 class="green">{{ subgroup }}</h1>
    <svg class="game">
      <line
        v-for="(line, index) of lines"
        :key="index"
        v-bind="line"
        stroke="red"
        :stroke-dasharray="line.strokeDasharray"
      ></line>
      <circle
        v-for="(point, index) of points"
        :key="index"
        :cx="point.x"
        :cy="point.y"
        r="5"
        :fill="point.fill"
        @click="select(index)"
      ></circle>
    </svg>
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
.game {
  height: 500px;
  user-select: none;
}

@media (min-width: 1024px) {
  .greetings h1,
  .greetings h3 {
    text-align: left;
  }
}
</style>
