<script setup lang="ts">
import { match } from "pinyin-pro";
import { useI18n } from "vue-i18n";
import { useRouter } from "vue-router";
import SearchResult from "./SearchResult.vue";
import SearchFooter from "./SearchFooter.vue";
import { useNav } from "@/layout/hooks/useNav";
import { transformI18n } from "@/plugins/i18n";
import { ref, computed, shallowRef } from "vue";
import { cloneDeep, isAllEmpty } from "@pureadmin/utils";
import { useDebounceFn, onKeyStroke } from "@vueuse/core";
import { usePermissionStoreHook } from "@/store/modules/permission";
import Search from "@iconify-icons/ri/search-line";
import { localForage } from "@/utils/localforage";
import { log } from "console";
import { watchEffect } from "vue";

interface Props {
  /** 弹窗显隐 */
  value: boolean;
}
interface optionsItem {
  path: string;
  meta?: {
    icon?: string;
    title?: string;
  };
}

interface Emits {
  (e: "update:value", val: boolean): void;
}

interface HistoryAndFavorite {
  history: optionsItem[];
  favorite: optionsItem[];
}

/**
 * 搜索模态框组件
 */

const { device } = useNav(); // 使用 useNav() 自定义 hook 获取设备信息
const emit = defineEmits<Emits>(); // 定义组件的自定义事件
const props = withDefaults(defineProps<Props>(), {}); // 使用 withDefaults() 函数设置组件的默认属性
const router = useRouter(); // 使用 useRouter() 自定义 hook 获取路由信息
const { locale } = useI18n(); // 使用 useI18n() 自定义 hook 获取国际化信息

const keyword = ref(""); // 创建响应式引用类型变量 keyword，用于存储搜索关键字
const scrollbarRef = ref(); // 创建响应式引用类型变量 scrollbarRef，用于存储滚动条的引用
const resultRef = ref(); // 创建响应式引用类型变量 resultRef，用于存储搜索结果的引用
const activePath = ref(""); // 创建响应式引用类型变量 activePath，用于存储当前激活的路径
const inputRef = ref<HTMLInputElement | null>(null); // 创建响应式引用类型变量 inputRef，用于存储输入框的引用
const resultOptions = shallowRef([]); // 创建响应式引用类型变量 resultOptions，用于存储搜索结果选项
const handleSearch = useDebounceFn(search, 300); // 使用 useDebounceFn() 自定义 hook 创建防抖函数 handleSearch，用于处理搜索操作
const historyAndFavoriteKey = "historyAndFavorite"; // 历史搜索记录和收藏菜单的 key
const historyAndFavorite = ref<HistoryAndFavorite | null>();
localForage()
  .getItem<HistoryAndFavorite | null>(historyAndFavoriteKey)
  .then(res => {
    historyAndFavorite.value = res;
  });

/** 菜单树形结构 */
const menusData = computed(() => {
  return cloneDeep(usePermissionStoreHook().wholeMenus);
});

const show = computed({
  get() {
    return props.value;
  },
  set(val: boolean) {
    emit("update:value", val);
  }
});

/** 将菜单树形结构扁平化为一维数组，用于菜单查询 */
function flatTree(arr) {
  const res = [];
  function deep(arr) {
    arr.forEach(item => {
      res.push(item);
      item.children && deep(item.children);
    });
  }
  deep(arr);
  return res;
}

/** 查询 */
function search() {
  const flatMenusData = flatTree(menusData.value);
  resultOptions.value = flatMenusData.filter(menu =>
    keyword.value
      ? transformI18n(menu.meta?.title)
          .toLocaleLowerCase()
          .includes(keyword.value.toLocaleLowerCase().trim()) ||
        (locale.value === "zh" &&
          !isAllEmpty(
            match(
              transformI18n(menu.meta?.title).toLocaleLowerCase(),
              keyword.value.toLocaleLowerCase().trim()
            )
          ))
      : false
  );
  if (resultOptions.value?.length > 0) {
    activePath.value = resultOptions.value[0].path;
  } else {
    activePath.value = "";
  }
}

function handleClose() {
  show.value = false;
  /** 延时处理防止用户看到某些操作 */
  setTimeout(() => {
    resultOptions.value = [];
    keyword.value = "";
  }, 200);
}

function scrollTo(index) {
  const scrollTop = resultRef.value.handleScroll(index);
  scrollbarRef.value.setScrollTop(scrollTop);
}

/** key up */
function handleUp() {
  const { length } = resultOptions.value;
  if (length === 0) return;
  const index = resultOptions.value.findIndex(
    item => item.path === activePath.value
  );
  if (index === 0) {
    activePath.value = resultOptions.value[length - 1].path;
    scrollTo(resultOptions.value.length - 1);
  } else {
    activePath.value = resultOptions.value[index - 1].path;
    scrollTo(index - 1);
  }
}

/** key down */
function handleDown() {
  const { length } = resultOptions.value;
  if (length === 0) return;
  const index = resultOptions.value.findIndex(
    item => item.path === activePath.value
  );
  if (index + 1 === length) {
    activePath.value = resultOptions.value[0].path;
  } else {
    activePath.value = resultOptions.value[index + 1].path;
  }
  scrollTo(index + 1);
}

function historyPush() {
  // 通过 resultOptions.value 中的 activePath.value 获取到当前选中的菜单项
  const options = [...resultOptions.value];
  const currentMenu = options
    .map(item => {
      return {
        path: item.path,
        meta: {
          icon: item.meta?.icon,
          title: item.meta?.title
        }
      };
    })
    .find(item => item.path === activePath.value);
  console.log("currentMenu", currentMenu);
  localForage()
    .getItem<HistoryAndFavorite>(historyAndFavoriteKey)
    .then(res => {
      console.log("res", res);

      if (res) {
        const { history, favorite } = res;
        const historyIndex = history.findIndex(
          item => item.path === currentMenu.path
        );
        const favoriteIndex = favorite.findIndex(
          item => item.path === currentMenu.path
        );
        if (~!historyIndex) {
          history.splice(historyIndex, 1);
        }
        if (~!favoriteIndex) {
          favorite.splice(favoriteIndex, 1);
        }
        history.unshift(currentMenu);
        const result = {
          history,
          favorite
        };
        historyAndFavorite.value = result;

        console.log("historyAndFavorite.value", historyAndFavorite.value);
        localForage().setItem(historyAndFavoriteKey, result);
      } else {
        console.log("res", res);
        const result = {
          history: [currentMenu],
          favorite: []
        };
        historyAndFavorite.value = result;
        localForage().setItem(historyAndFavoriteKey, result);
      }

      console.log(
        "最终结果",
        localForage()
          .getItem(historyAndFavoriteKey)
          .then(res => console.log(historyAndFavorite.value))
      );
    });
}

/** key enter */ // TODO 2. 通过 localforage 将选中的菜单项存储到本地 分为两步：历史搜索记录和收藏菜单
function handleEnter() {
  const { length } = resultOptions.value;
  if (length === 0 || activePath.value === "") return;
  console.log("activePath.value", activePath.value);
  console.log("result", resultOptions.value);

  // 2.1. historyAndFavorite 对象中的 history 数组中是否已经存在 activePath.value
  // console.log("historyAndFavorite", historyAndFavorite);
  historyPush();
  // router.push(activePath.value);
  // handleClose();
}

onKeyStroke("Enter", handleEnter);
onKeyStroke("ArrowUp", handleUp);
onKeyStroke("ArrowDown", handleDown);
</script>

<template>
  <el-dialog
    v-model="show"
    top="5vh"
    class="pure-search-dialog"
    :show-close="false"
    :width="device === 'mobile' ? '80vw' : '40vw'"
    :before-close="handleClose"
    :style="{
      borderRadius: '6px'
    }"
    append-to-body
    @opened="inputRef.focus()"
    @closed="inputRef.blur()"
  >
    <el-input
      ref="inputRef"
      v-model="keyword"
      size="large"
      clearable
      placeholder="搜索菜单（中文模式下支持拼音搜索）"
      @input="handleSearch"
    >
      <template #prefix>
        <IconifyIconOffline
          :icon="Search"
          class="text-primary w-[24px] h-[24px]"
        />
      </template>
    </el-input>
    <div class="search-result-container">
      <el-scrollbar ref="scrollbarRef" max-height="calc(90vh - 140px)">
        <el-empty
          v-if="resultOptions.length === 0"
          description="暂无搜索结果"
        />
        <SearchResult
          v-else
          ref="resultRef"
          v-model:value="activePath"
          :options="resultOptions"
          :historyAndFavorite="historyAndFavorite"
          @click="handleEnter"
        />
      </el-scrollbar>
    </div>
    <template #footer>
      <SearchFooter :total="resultOptions.length" />
    </template>
  </el-dialog>
</template>

<style lang="scss" scoped>
.search-result-container {
  margin-top: 12px;
}
</style>
