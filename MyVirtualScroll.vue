<template>
  <div class="virtual-scroll" ref="scrollContainer">
    <q-scroll-observer @scroll="onScroll" :debounce="debounce" />
    <div class="virtual-scroll-content" ref="scrollContent">
      <div class="virtual-filler-top" :style="{ height: `${topFillerHeight}px` }" />
      <template v-for="(item, index) in items" :key="getItemKey(item)">
        <div class="virtual-item" v-if="showItem(index) != 'deleted'" v-show="showItem(index) != 'hidden'">
          <q-resize-observer @resize="(event) => onItemHeightChange(index, event)" />
          <slot :item="item" :index="index" />
        </div>
      </template>
      <div class="virtual-filler-bottom" :style="{ height: `${bottomFillerHeight}px` }" />
    </div>
    <div
      class="virtual-scroll-loading"
      :class="{ invisible: !(showLoading && isLoading) }"
      v-if="!stopInfiniteLoad && infiniteLoadType === 'bottom'"
      ref="loadingRef"
    >
      <div class="row justify-center q-my-md">
        <slot name="loading" />
      </div>
    </div>
  </div>
</template>

<script setup lang="ts" generic="T">
// NOTE: I didn't add infinite load type "top" because there's many problems with it
import { computed, nextTick, onMounted, onUnmounted, ref, watch } from "vue";

interface Props {
  items: T[];
  margins?: number;
  offset?: number;
  bottomOffset?: number;
  debounce?: number;
  itemKey: keyof T;
  infiniteLoadType?: "bottom" | "none";
}

const props = withDefaults(defineProps<Props>(), {
  margins: 0,
  offset: 250,
  bottomOffset: 500,
  debounce: 150
});

const emit = defineEmits<{
  (e: "loadMore", index: number, done: (stop?: boolean) => void): void;
}>();

watch(
  props.items,
  () => {
    if (!isLoading.value) updateVisibleItems();
  },
  { deep: true }
);

const heights = ref<number[]>([]);
const containerHeight = ref(0);

const scrollContent = ref<HTMLElement | null>(null);
const scrollContainer = ref<HTMLElement | null>(null);
const loadingRef = ref<HTMLElement | null>(null);

const top = ref(0);
const bottom = computed(() => top.value + containerHeight.value + props.offset + props.bottomOffset);

const topFillerHeight = ref(0);
const bottomFillerHeight = ref(0);

const visibleIndexes = ref<number[]>([]);

const isLoading = ref(false);
const stopInfiniteLoad = ref(false);

let isLoadingTimeout: NodeJS.Timeout | undefined = undefined;
const showLoading = ref(false);

function updateSvgAnimations(isRetry?: boolean) {
  if (renderLoadingSlot.value === true) {
    if (loadingRef.value === null) {
      isRetry !== true &&
        nextTick(() => {
          updateSvgAnimations(true);
        });
      return;
    }

    const isVisible = isLoading.value && showLoading.value;

    const action = `${isVisible === true ? "un" : ""}pauseAnimations` as const;
    Array.from(loadingRef.value.getElementsByTagName("svg")).forEach((el) => {
      el[action]();
    });
  }
}

const renderLoadingSlot = computed(() => props.infiniteLoadType === "bottom" && !stopInfiniteLoad.value === true);

watch([isLoading, renderLoadingSlot], () => {
  updateSvgAnimations();
});

watch(
  isLoading,
  () => {
    checkLoading();
  },
  { immediate: true }
);

// eslint-disable-next-line @typescript-eslint/no-explicit-any
const getItemKey = (item: any) => {
  return item[props.itemKey];
};

function checkLoading() {
  if (isLoading.value || stopInfiniteLoad.value) return;

  const loadMore = () => {
    isLoading.value = true;

    emit("loadMore", props.items.length - 1, (stop?: boolean) => {
      isLoading.value = false;
      if (stop) {
        stopInfiniteLoad.value = true;
      }
      if (isLoadingTimeout) {
        clearTimeout(isLoadingTimeout);
        isLoadingTimeout = undefined;
      }
      if (showLoading.value) {
        isLoadingTimeout = setTimeout(() => {
          showLoading.value = false;
          isLoadingTimeout = undefined;
        }, 250);
      }
      updateVisibleItems();
    });
  };

  if (props.infiniteLoadType === "bottom" && bottom.value >= (scrollContent.value?.clientHeight || 0)) {
    if (isLoadingTimeout) {
      clearTimeout(isLoadingTimeout);
      isLoadingTimeout = undefined;
    }
    isLoadingTimeout = setTimeout(() => {
      showLoading.value = true;
      isLoadingTimeout = undefined;
    }, 250);
    loadMore();
  }
}

function updateVisibleItems() {
  nextTick(() => {
    if (!scrollContent.value?.checkVisibility()) return;
    let cumulativeHeight = 0;
    let topIndex = -1;
    let bottomIndex = -1;

    for (let i = 0; i < heights.value.length; i++) {
      const itemHeight = heights.value[i] || 0;
      const nextCumulativeHeight = cumulativeHeight + itemHeight;

      if (topIndex === -1 && top.value < nextCumulativeHeight) {
        topIndex = i;
      }

      if (bottom.value <= nextCumulativeHeight) {
        bottomIndex = i;
        break;
      }

      cumulativeHeight = nextCumulativeHeight;
    }

    if (bottomIndex === -1) {
      bottomIndex = heights.value.length - 1;
    }

    visibleIndexes.value = [topIndex, bottomIndex];

    topFillerHeight.value = heights.value.slice(0, topIndex).reduce((acc, height) => acc + (height || 0), 0);
    bottomFillerHeight.value = heights.value.slice(bottomIndex + 1).reduce((acc, height) => acc + (height || 0), 0);

    checkLoading();
  });
}

function onScroll(info: QScrollObserverDetails) {
  top.value = Math.max(info.position.top - props.offset, 0);
  updateVisibleItems();
}

function onResize() {
  containerHeight.value = window.innerHeight;
  updateVisibleItems();
}

function onItemHeightChange(index: number, info: { height: number; width: number }) {
  heights.value[index] = info.height + (props.margins || 0);
  updateVisibleItems();
}

function hasHeight(index: number) {
  return !!heights.value[index];
}

function showItem(index: number) {
  if (index >= visibleIndexes.value[0] && index <= visibleIndexes.value[1]) return "visible";
  else if (!hasHeight(index)) return "hidden";
  else return "deleted";
}

onMounted(() => {
  nextTick(() => {
    if (scrollContainer.value) {
      containerHeight.value = window.innerHeight;
    }
    updateVisibleItems();
  });
});

onMounted(() => {
  window.addEventListener("resize", onResize);

  onResize();
});

onUnmounted(() => {
  window.removeEventListener("resize", onResize);
});
</script>
