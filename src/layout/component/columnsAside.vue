<template>
  <div class="layout-columns-aside my-flex-column">
    <div v-if="setShowLogo" class="layout-logo">
      <img :src="logoMini" class="layout-logo-medium-img" />
    </div>
    <el-scrollbar class="my-flex-fill">
      <ul @mouseleave="onColumnsAsideMenuMouseleave()">
        <li
          v-for="(v, k) in state.columnsAsideList"
          :key="k"
          @click="onColumnsAsideMenuClick(v, k)"
          @mouseenter="onColumnsAsideMenuMouseenter(v, k)"
          :ref="
            (el) => {
              if (el) columnsAsideOffsetTopRefs[k] = el
            }
          "
          :class="{ 'layout-columns-active': state.liIndex === k, 'layout-columns-hover': state.liHoverIndex === k }"
          :title="$t(v.meta.title)"
        >
          <div :class="themeConfig.columnsAsideLayout" v-if="!v.meta.isLink || (v.meta.isLink && v.meta.isIframe)">
            <SvgIcon :name="v.meta.icon" />
            <div class="columns-vertical-title font12">
              {{
                $t(v.meta.title) && $t(v.meta.title).length >= 4
                  ? $t(v.meta.title).substr(0, themeConfig.columnsAsideLayout === 'columns-vertical' ? 4 : 3)
                  : $t(v.meta.title)
              }}
            </div>
          </div>
          <div :class="themeConfig.columnsAsideLayout" v-else>
            <a :href="v.meta.isLink" target="_blank">
              <SvgIcon :name="v.meta.icon" />
              <div class="columns-vertical-title font12">
                {{
                  $t(v.meta.title) && $t(v.meta.title).length >= 4
                    ? $t(v.meta.title).substr(0, themeConfig.columnsAsideLayout === 'columns-vertical' ? 4 : 3)
                    : $t(v.meta.title)
                }}
              </div>
            </a>
          </div>
        </li>
        <div ref="columnsAsideActiveRef" :class="themeConfig.columnsAsideStyle"></div>
      </ul>
    </el-scrollbar>
  </div>
</template>

<script setup lang="ts" name="layoutColumnsAside">
import { reactive, ref, onMounted, nextTick, watch, onUnmounted, computed } from 'vue'
import { useRoute, useRouter, onBeforeRouteUpdate, RouteRecordRaw } from 'vue-router'
import { storeToRefs } from 'pinia'
import pinia from '/@/stores/index'
import { useRoutesList } from '/@/stores/routesList'
import { useThemeConfig } from '/@/stores/themeConfig'
import mittBus from '/@/utils/mitt'
import logoMini from '/@/assets/logo-mini.svg'

// ??????????????????
const columnsAsideOffsetTopRefs = ref<RefType>([])
const columnsAsideActiveRef = ref()
const stores = useRoutesList()
const storesThemeConfig = useThemeConfig()
const { routesList, isColumnsMenuHover, isColumnsNavHover } = storeToRefs(stores)
const { themeConfig } = storeToRefs(storesThemeConfig)
const route = useRoute()
const router = useRouter()
const state = reactive<ColumnsAsideState>({
  columnsAsideList: [],
  liIndex: 0,
  liOldIndex: null,
  liHoverIndex: null,
  liOldPath: null,
  difference: 0,
  routeSplit: [],
})

// ????????????/?????? logo
const setShowLogo = computed(() => {
  let { layout, isShowLogo } = themeConfig.value
  return (isShowLogo && layout === 'defaults') || (isShowLogo && layout === 'columns')
})

// ??????????????????????????????
const setColumnsAsideMove = (k: number) => {
  state.liIndex = k
  columnsAsideActiveRef.value.style.top = `${columnsAsideOffsetTopRefs.value[k].offsetTop + state.difference}px`
}
// ????????????????????????
const onColumnsAsideMenuClick = (v: RouteItem, k: number) => {
  setColumnsAsideMove(k)
  let { path, redirect } = v
  if (redirect) router.push(redirect)
  else router.push(path)
}
// ?????????????????????????????????????????????
const onColumnsAsideMenuMouseenter = (v: RouteRecordRaw, k: number) => {
  if (!themeConfig.value.isColumnsMenuHoverPreload) return false
  let { path } = v
  state.liOldPath = path
  state.liOldIndex = k
  state.liHoverIndex = k
  mittBus.emit('setSendColumnsChildren', setSendChildren(path))
  stores.setColumnsMenuHover(false)
  stores.setColumnsNavHover(true)
}
// ?????????????????????????????????????????????
const onColumnsAsideMenuMouseleave = async () => {
  await stores.setColumnsNavHover(false)
  // ????????????????????????????????? store.state.routesList ??????????????????
  setTimeout(() => {
    if (!isColumnsMenuHover && !isColumnsNavHover) mittBus.emit('restoreDefault')
  }, 100)
}
// ????????????????????????
const onColumnsAsideDown = (k: number) => {
  nextTick(() => {
    setColumnsAsideMove(k)
  })
}
// ??????/??????????????????????????????/???????????????????????????
const setFilterRoutes = () => {
  state.columnsAsideList = filterRoutesFun(routesList.value)
  const resData: MittMenu = setSendChildren(route.path)
  if (Object.keys(resData).length <= 0) return false
  onColumnsAsideDown(resData.item?.k)
  mittBus.emit('setSendColumnsChildren', resData)
}
// ????????????????????????????????????
const setSendChildren = (path: string) => {
  const currentPathSplit = path.split('/')
  let currentData: MittMenu = { children: [] }
  state.columnsAsideList.map((v: RouteItem, k: number) => {
    if (v.path === `/${currentPathSplit[1]}`) {
      v['k'] = k
      currentData['item'] = { ...v }
      currentData['children'] = [{ ...v }]
      if (v.children) currentData['children'] = v.children
    }
  })
  return currentData
}
// ????????????????????????
const filterRoutesFun = <T extends RouteItem>(arr: T[]): T[] => {
  return arr
    .filter((item: T) => !item.meta?.isHide)
    .map((item: T) => {
      item = Object.assign({}, item)
      if (item.children) item.children = filterRoutesFun(item.children)
      return item
    })
}
// tagsView ???????????????????????????????????? columnsAsideList???????????????????????????
const setColumnsMenuHighlight = (path: string) => {
  state.routeSplit = path.split('/')
  state.routeSplit.shift()
  const routeFirst = `/${state.routeSplit[0]}`
  const currentSplitRoute = state.columnsAsideList.find((v: RouteItem) => v.path === routeFirst)
  if (!currentSplitRoute) return false
  // ??????????????????????????????
  setTimeout(() => {
    onColumnsAsideDown(currentSplitRoute.k)
  }, 0)
}
// ???????????????
onMounted(() => {
  setFilterRoutes()
  // ?????????????????????????????????????????????????????????????????????
  mittBus.on('restoreDefault', () => {
    state.liOldIndex = null
    state.liOldPath = null
  })
})
// ???????????????
onUnmounted(() => {
  mittBus.off('restoreDefault', () => {})
})
// ???????????????
onBeforeRouteUpdate((to) => {
  setColumnsMenuHighlight(to.path)
  mittBus.emit('setSendColumnsChildren', setSendChildren(to.path))
})
// ??????????????????????????????????????????????????????????????????????????????
watch(
  pinia.state,
  (val) => {
    val.themeConfig.themeConfig.columnsAsideStyle === 'columnsRound' ? (state.difference = 3) : (state.difference = 0)
    if (!val.routesList.isColumnsMenuHover && !val.routesList.isColumnsNavHover) {
      state.liHoverIndex = null
      mittBus.emit('setSendColumnsChildren', setSendChildren(route.path))
    } else {
      state.liHoverIndex = state.liOldIndex
      if (!state.liOldPath) return false
      mittBus.emit('setSendColumnsChildren', setSendChildren(state.liOldPath))
    }
  },
  {
    deep: true,
  }
)
</script>

<style scoped lang="scss">
.layout-columns-aside {
  width: 70px;
  height: 100%;
  background: var(--next-bg-columnsMenuBar);
  ul {
    position: relative;
    .layout-columns-active {
      color: var(--next-color-columnsMenuBarActiveColor) !important;
      transition: 0.3s ease-in-out;
    }
    .layout-columns-hover {
      color: var(--el-color-primary);
      a {
        color: var(--el-color-primary);
      }
    }
    li {
      color: var(--next-bg-columnsMenuBarColor);
      width: 100%;
      height: 50px;
      text-align: center;
      display: flex;
      cursor: pointer;
      position: relative;
      z-index: 1;
      &:hover {
        @extend .layout-columns-hover;
      }
      .columns-vertical {
        margin: auto;
        .columns-vertical-title {
          padding-top: 1px;
        }
      }
      .columns-horizontal {
        display: flex;
        height: 50px;
        width: 100%;
        align-items: center;
        padding: 0 5px;
        i {
          margin-right: 3px;
        }
        a {
          display: flex;
          .columns-horizontal-title {
            padding-top: 1px;
          }
        }
      }
      a {
        text-decoration: none;
        color: var(--next-bg-columnsMenuBarColor);
      }
    }
    .columns-round {
      background: var(--el-color-primary);
      color: var(--el-color-white);
      position: absolute;
      left: 50%;
      top: 2px;
      height: 44px;
      width: 65px;
      transform: translateX(-50%);
      z-index: 0;
      transition: 0.3s ease-in-out;
      border-radius: 5px;
    }
    .columns-card {
      @extend .columns-round;
      top: 0;
      height: 50px;
      width: 100%;
      border-radius: 0;
    }
  }
}

.layout-logo {
  height: 50px;
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: rgb(0 21 41 / 2%) 0px 1px 4px;
  &-medium-img {
    width: 30px;
  }
}
</style>
