<script setup lang="ts">
// @ts-ignore
import Algebra from "ganja.js";
import {
  dot,
  gcd,
  getSingleCommaName,
  inverseLogMetric,
  mmod,
  natsToCents,
  patentVal,
  PRIMES,
  Temperament,
} from "temperaments";
import { toWarts } from "temperaments/dist/src/temperament";
import { computed, onMounted, onUnmounted, reactive, ref, watch } from "vue";

const props = defineProps<{
  subgroup: number[];
  showWarts: boolean;
}>();

const subgroupStr = computed(() => {
  return props.subgroup.map((i) => PRIMES[i]).join(".");
});

const zoomLevel = ref(6000);
const centerX = ref(0);
const centerY = ref(0);

function onMouseWheel(event_: Event) {
  const event: WheelEvent = event_ as WheelEvent;
  if (event.deltaY < 0) {
    increaseZoomLevel();
  }
  if (event.deltaY > 0) {
    decreaseZoomLevel();
  }
}

function onDOMMouseScroll(event: any) {
  if (event.detail < 0) {
    increaseZoomLevel();
  }
  if (event.detail > 0) {
    decreaseZoomLevel();
  }
}

function increaseZoomLevel() {
  zoomLevel.value *= 1.1;
}

function decreaseZoomLevel() {
  zoomLevel.value /= 1.1;
}

const dragStart = {
  x: 0,
  y: 0,
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
  const deltaX = dragStart.screenX - event.screenX;
  const deltaY = dragStart.screenY - event.screenY;
  // TODO: Figure out why the screen jerks
  // if (Math.hypot(deltaX, deltaY) < 5) {
  //   return;
  // }
  centerX.value = dragStart.x + deltaX / zoomLevel.value;
  centerY.value = dragStart.y + deltaY / zoomLevel.value;
}

function onMouseUp(event: MouseEvent) {
  onMouseMove(event);
  dragStart.active = false;
}

//FF doesn't recognize mousewheel as of FF3.x
const mousewheelevt = /Firefox/i.test(navigator.userAgent)
  ? "DOMMouseScroll"
  : "mousewheel";
const onMouseWheelEvt = /Firefox/i.test(navigator.userAgent)
  ? onDOMMouseScroll
  : onMouseWheel;

onMounted(() => {
  if ("attachEvent" in document)
    //if IE (and Opera depending on user setting)
    (document as any).attachEvent("on" + mousewheelevt, onMouseWheel);
  else if (document.addEventListener)
    //WC3 browsers
    document.addEventListener(mousewheelevt, onMouseWheelEvt, false);
  window.addEventListener("mousedown", onMouseDown);
  window.addEventListener("mousemove", onMouseMove);
  window.addEventListener("mouseup", onMouseUp);
});

onUnmounted(() => {
  if ("detachEvent" in document)
    //if IE (and Opera depending on user setting)
    (document as any).detachEvent("on" + mousewheelevt, onMouseWheel);
  else if (document.removeEventListener)
    //WC3 browsers
    document.removeEventListener(mousewheelevt, onMouseWheelEvt, false);
  window.removeEventListener("mousedown", onMouseDown);
  window.removeEventListener("mousemove", onMouseMove);
  window.removeEventListener("mouseup", onMouseUp);
});

const Cl3 = Algebra(3);

const selected = ref(-1);
const mouseOverLabel = reactive({
  text: "N/A",
  show: false,
  x: 0,
  y: 0,
  index: -1,
});
const temperamentName = ref("Ya");

const vals: number[][] = reactive([]);
const valPairs: number[][] = reactive([]);

watch(
  props.subgroup,
  (newValue) => {
    const numComponents = newValue.reduce((a, b) => Math.max(a, b)) + 1;
    const edo3 = patentVal(3, 2, numComponents);
    const edo4 = patentVal(4, 2, numComponents);
    const edo5 = patentVal(5, 2, numComponents);
    const middle = patentVal(15 - numComponents, 2, numComponents);

    while (vals.length) {
      vals.pop();
    }
    vals.push(edo3);
    vals.push(edo4);
    vals.push(edo5);
    vals.push(middle);

    while (valPairs.length) {
      valPairs.pop();
    }

    valPairs.push([0, 1]);
    valPairs.push([0, 2]);
    valPairs.push([1, 2]);

    zoomLevel.value = 6000;
    centerX.value = 0;
    centerY.value = 0;
    selected.value = -1;
    if (numComponents === 3) {
      temperamentName.value = "Ya";
    } else {
      if (newValue[1] === 2) {
        temperamentName.value = "Yaza";
      } else {
        temperamentName.value = "Za";
      }
    }
  },
  { immediate: true }
);

const diagonalYScale = Math.sqrt(0.75);

const screenCenterX = 450;
const screenCenterY = 270;

const points = computed(() => {
  const metric = inverseLogMetric(props.subgroup);
  return vals.map((val, index) => {
    const sg = props.subgroup;
    let a = val[sg[0]] * metric[0];
    let b = val[sg[1]] * metric[1];
    let c = val[sg[2]] * metric[2];
    // This is the projective norm. Works correctly even if vals get flipped.
    const normalizer = 1 / (a + b + c);

    a *= normalizer;
    b *= normalizer;
    c *= normalizer;
    const x = 0.5 * (a + b) - c;
    const y = (a - b) * diagonalYScale;
    return {
      x: (x - centerX.value) * zoomLevel.value + screenCenterX,
      y: (y - centerY.value) * zoomLevel.value + screenCenterY,
      fill: index === selected.value ? "green" : "black",
    };
  });
});

const center = computed(() => {
  return {
    x: screenCenterX - centerX.value * zoomLevel.value,
    y: screenCenterY - centerY.value * zoomLevel.value,
  };
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

function cleanVal(val: number[]): number[] {
  const commonFactor = val.reduce(gcd);
  for (let k = 0; k < val.length; ++k) {
    val[k] /= commonFactor;
  }
  const norm = val[0] + val[1] + val[2];
  if (norm < 0) {
    for (let k = 0; k < val.length; ++k) {
      val[k] = -val[k];
    }
  }
  const numComponents = props.subgroup.reduce((a, b) => Math.max(a, b)) + 1;
  const result = patentVal(val[0], 2, numComponents);
  props.subgroup.forEach((index, i) => (result[index] = val[i]));
  return result;
}

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

const te = reactive({
  period: 1200.0,
  generator: (Math.log(3) / Math.LN2) * 1200.0,
});
const pote = reactive({
  period: 1200.0,
  generator: (Math.log(1.5) / Math.LN2) * 1200.0,
});

function nameTemperament(pair: [number, number]) {
  const sg = props.subgroup;
  const temperament = Temperament.fromValList(
    [vals[pair[0]], vals[pair[1]]],
    sg
  );
  if (temperament.isNil()) {
    temperamentName.value = "Nil";
    te.period = 0;
    te.generator = 0;
    pote.period = 0;
    pote.generator = 0;
    return;
  }
  temperament.canonize();
  const name = getSingleCommaName(temperament);
  temperamentName.value = name.given || name.color || "???";
  const tuning = temperament.toTenneyEuclid();
  let [divisions, generator] = temperament.divisionsGenerator();
  if (divisions !== 0) {
    te.period = natsToCents(tuning[0] / divisions);
    te.generator = Math.min(
      mmod(natsToCents(dot(tuning, generator)), te.period),
      mmod(-natsToCents(dot(tuning, generator)), te.period)
    );
    pote.period = 1200 / Math.abs(divisions);
    pote.generator = Math.abs((Math.LN2 / tuning[0]) * te.generator);
  } else {
    // Tempered out the octave, we're in 3.5 now.
    divisions = temperament.value[6];
    te.period = natsToCents(Math.log(3) / divisions);
    te.generator = natsToCents(Math.log(5 / 3));
    pote.period = te.period;
    pote.generator = Math.abs(te.generator);
  }
}

function checkIntersections() {
  const sg = props.subgroup;

  const vectors = vals.map((val_) => {
    const val = sg.map((i) => val_[i]);
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
      const val = [intersection[1], intersection[2], intersection[3]];
      if (!val[0] && !val[1] && !val[2]) {
        continue;
      }
      const v1 = cleanVal(val);
      let novel = true;
      vals.forEach((v) => {
        if (
          v[sg[0]] === v1[sg[0]] &&
          v[sg[1]] === v1[sg[1]] &&
          v[sg[2]] === v1[sg[2]]
        ) {
          novel = false;
        }
      });
      if (novel) {
        vals.push(v1);
      }
    }
  }
}

function setMouseOverLabel(index: number) {
  if (props.showWarts) {
    mouseOverLabel.text = toWarts(vals[index]);
  } else {
    mouseOverLabel.text = vals[index][0].toString();
  }
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
    <h1 class="green">{{ subgroupStr }} : {{ temperamentName }}</h1>
    <svg class="game">
      <rect
        :x="center.x - 2.5"
        :y="center.y - 2.5"
        width="5"
        height="5"
        stroke="blue"
        fill-opacity="0"
      ></rect>
      <rect
        :x="center.x - 2.5"
        :y="center.y - 2.5"
        width="5"
        height="5"
        stroke="blue"
        fill-opacity="0"
        :transform="'rotate(45,' + center.x + ',' + center.y + ')'"
      ></rect>
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
