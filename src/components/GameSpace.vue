<script setup lang="ts">
// @ts-ignore
import Algebra from "ganja.js";
import {
  dot,
  gcd,
  getRank2Name,
  inverseLogMetric,
  mmod,
  natsToCents,
  patentVal,
  Temperament,
} from "temperaments";
import { computed, onMounted, onUnmounted, reactive, ref } from "vue";

defineProps<{
  subgroup: string;
}>();

const zoomLevel = ref(6000);
const centerX = ref(300);
const centerY = ref(300);

// TODO: Compensate for center dragging
function onMouseWheel(event_: Event) {
  const event: WheelEvent = event_ as WheelEvent;
  if (event.deltaY < 0) {
    zoomLevel.value *= 1.1;
  }
  if (event.deltaY > 0) {
    zoomLevel.value /= 1.1;
  }
}

const dragStart = {
  x: 300,
  y: 300,
  screenX: 0,
  screenY: 0,
  active: false,
};

function onMouseDown(event: MouseEvent) {
  dragStart.x = centerX.value;
  dragStart.y = centerY.value;
  dragStart.screenX = event.screenX;
  dragStart.screenY = event.screenY;
  dragStart.active = true;
}

function onMouseMove(event: MouseEvent) {
  if (!dragStart.active) {
    return;
  }
  centerX.value = dragStart.x - dragStart.screenX + event.screenX;
  centerY.value = dragStart.y - dragStart.screenY + event.screenY;
}

function onMouseUp(event: MouseEvent) {
  onMouseMove(event);
  dragStart.active = false;
}

onMounted(() => {
  window.addEventListener("mousewheel", onMouseWheel);
  window.addEventListener("mousedown", onMouseDown);
  window.addEventListener("mousemove", onMouseMove);
  window.addEventListener("mouseup", onMouseUp);
});

onUnmounted(() => {
  window.removeEventListener("mousewheel", onMouseWheel);
  window.removeEventListener("mousedown", onMouseDown);
  window.removeEventListener("mousemove", onMouseMove);
  window.removeEventListener("mouseup", onMouseUp);
});

const Cl3 = Algebra(3);

const edo3 = patentVal(3, 2, 3);
const edo4 = patentVal(4, 2, 3);
const edo5 = patentVal(5, 2, 3);
const edo12 = patentVal(12, 2, 3);

const vals = reactive([edo3, edo4, edo5, edo12]);
const selected = ref(-1);
const mouseOverLabel = reactive({
  text: "N/A",
  show: false,
  x: 0,
  y: 0,
  index: -1,
});

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
      x: x * zoomLevel.value + centerX.value,
      y: y * zoomLevel.value + centerY.value,
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
    const pair: [number, number] = [selected.value, index];
    nameTemperament(pair);
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

const temperamentName = ref("Just Intonation");
const te = reactive({
  period: 1200.0,
  generator: (Math.log(3) / Math.LN2) * 1200.0,
});
const pote = reactive({
  period: 1200.0,
  generator: (Math.log(3) / Math.LN2) * 1200.0,
});

function nameTemperament(pair: [number, number]) {
  const sg = [0, 1, 2];
  const temperament = Temperament.fromValList(
    [vals[pair[0]], vals[pair[1]]],
    sg
  );
  temperament.canonize();
  const prefix = temperament.rank2Prefix();
  const recovered = Temperament.recoverRank2(prefix, sg);
  recovered.canonize();
  if (temperament.equals(recovered)) {
    const maybeName = getRank2Name(sg, prefix);
    if (maybeName !== undefined) {
      temperamentName.value = maybeName;
    } else {
      temperamentName.value = "???";
    }
  }
  const tuning = temperament.toTenneyEuclid();
  const [divisions, generator] = temperament.divisionsGenerator();
  te.period = natsToCents(tuning[0] / divisions);
  te.generator = mmod(natsToCents(dot(tuning, generator)), te.period);
  pote.period = 1200 / divisions;
  pote.generator = (te.generator / tuning[0]) * Math.LN2;
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

function setMouseOverLabel(index: number) {
  mouseOverLabel.text = vals[index][0].toString();
  mouseOverLabel.x = points.value[index].x + 5;
  mouseOverLabel.y = points.value[index].y - 5;
  mouseOverLabel.index = index;
  mouseOverLabel.show = true;
}

function clearMouseOverLabel(index: number) {
  if (index === mouseOverLabel.index) {
    mouseOverLabel.show = false;
  }
}
</script>

<template>
  <div class="greetings">
    <h1 class="green">{{ subgroup }} : {{ temperamentName }}</h1>
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
        @mouseenter="setMouseOverLabel(index)"
        @mouseleave="clearMouseOverLabel(index)"
      ></circle>
      <text
        font-size="40px"
        :x="mouseOverLabel.x"
        :y="mouseOverLabel.y"
        v-if="mouseOverLabel.show"
        fill="black"
      >
        {{ mouseOverLabel.text }}
      </text>
    </svg>
    <p>
      POTE period={{ pote.period }} POTE generator={{ pote.generator }} TE
      period={{ te.period }} TE generator={{ te.generator }}
    </p>
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
  height: 60vh;
  width: 100%;
  user-select: none;
}

@media (min-width: 1024px) {
  .greetings h1,
  .greetings h3 {
    text-align: left;
  }
}
</style>
