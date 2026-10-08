<script setup lang="ts">
/**
 * Limooo 页头 —— 结构/样式对齐主站 Flask/src/templates/base.html 的 nav.site-nav。
 *
 * 这是 fork 定制件：不使用 VitePress 自带的 VPNavBar。
 * 品牌字「Limooo」用 Baloo 2，不放 logo 图片。
 */
import { computed, onBeforeUnmount, onMounted, ref } from 'vue'
import { useData, useRoute, useRouter } from 'vitepress'

import VPIcon from './VPIcon.vue'

interface LangItem {
  code: string
  label: string
  flag?: string
  default?: boolean
}

const { theme, isDark } = useData()
const route = useRoute()
const router = useRouter()

const limooo = computed<any>(() => (theme.value as any).limooo ?? {})
const languages = computed<LangItem[]>(() => limooo.value.languages ?? [])
const defaultLang = computed(
  () =>
    languages.value.find((l) => l.default)?.code ??
    languages.value[0]?.code ??
    ''
)

const langCodes = computed(() => languages.value.map((l) => l.code))

/**
 * themeConfig.limooo.langSegmentAlways：内容页路径是否**总是**带语言段
 * （含默认语言，如 /video-platform/zh-cn）。默认 false = 旧行为（默认语言无后缀）。
 * 首页永远是例外：`/` 就是默认语言。
 */
const langSegmentAlways = computed(
  () => (limooo.value as any).langSegmentAlways === true
)

/** 语言码是路径的最后一段：/video-platform/en-us；默认语言无后缀。 */
function langOfPath(path: string): string {
  const segments = path.split('?')[0].split('/').filter(Boolean)
  const last = segments[segments.length - 1]
  return last && langCodes.value.includes(last) ? last : defaultLang.value
}

const currentLang = computed(() => langOfPath(route.path))

function pathForLang(code: string): string {
  const segments = route.path.split('?')[0].split('/').filter(Boolean)
  if (segments.length && langCodes.value.includes(segments[segments.length - 1])) {
    segments.pop()
  }
  const base = segments.length ? '/' + segments.join('/') : '/'
  // 首页：`/` 是默认语言，其它语言是 /en-us、/ja-jp…
  if (base === '/') return code === defaultLang.value ? '/' : `/${code}`
  // 内容页：默认语言要不要后缀由 langSegmentAlways 决定
  if (code === defaultLang.value && !langSegmentAlways.value) return base
  return `${base}/${code}`
}

const brandLink = computed(() => {
  const code = currentLang.value
  return code === defaultLang.value ? '/' : `/${code}`
})

interface NavItem {
  text: string
  link: string
  activeMatch?: string
}

const navItems = computed<NavItem[]>(() => {
  const nav = (theme.value as any).nav
  if (!Array.isArray(nav)) return []
  return nav
    .filter((item: any) => item && item.link)
    .map((item: any) => ({
      text: item.text,
      link: item.link,
      activeMatch: item.activeMatch
    }))
})

function isActive(item: NavItem): boolean {
  const path = route.path
  const link = item.link.split('?')[0]
  if (item.activeMatch) return new RegExp(item.activeMatch).test(path)
  if (/^https?:/.test(link)) return false
  if (link === '/') return path === '/' || path === `/${currentLang.value}`
  return path === link || path.startsWith(link + '/')
}

function isExternal(link: string): boolean {
  return /^https?:\/\//.test(link)
}

const socialLinks = computed<any[]>(() => (theme.value as any).socialLinks ?? [])

const menuOpen = ref(false)
const langRoot = ref<HTMLElement | null>(null)

function toggleAppearance() {
  isDark.value = !isDark.value
}

/* 桌面端悬停展开/收起（与主站 base.js 的 bindLangHover 一致），
   触摸端保持点击切换。 */
const HOVER_QUERY = '(hover: hover) and (pointer: fine)'

function canHover(): boolean {
  return (
    typeof window !== 'undefined' && window.matchMedia(HOVER_QUERY).matches
  )
}

function onLangEnter() {
  if (canHover()) menuOpen.value = true
}

function onLangLeave() {
  if (canHover()) menuOpen.value = false
}

function toggleMenu() {
  menuOpen.value = !menuOpen.value
}

function closeMenu() {
  menuOpen.value = false
}

function onDocumentClick(event: MouseEvent) {
  if (menuOpen.value && langRoot.value && !langRoot.value.contains(event.target as Node)) {
    closeMenu()
  }
}

function goLang(code: string) {
  closeMenu()
  const target = pathForLang(code)
  if (target !== route.path) router.go(target)
}

onMounted(() => document.addEventListener('click', onDocumentClick))
onBeforeUnmount(() => document.removeEventListener('click', onDocumentClick))
</script>

<template>
  <div class="VPLimoooNav">
    <div class="nav-inner">
      <a :href="brandLink" class="nav-brand-link" aria-label="Limooo">
        <span class="nav-brand">Limooo</span>
      </a>

      <div class="nav-right">
        <div v-if="navItems.length" class="global-nav">
          <a
            v-for="item in navItems"
            :key="item.link"
            :href="item.link"
            class="nav-link"
            :class="{ active: isActive(item) }"
          >
            <span>{{ item.text }}</span>
            <svg
              v-if="isExternal(item.link)"
              class="nav-external"
              viewBox="0 0 24 24"
              fill="currentColor"
              aria-hidden="true"
            >
              <path d="M0 0h24v24H0V0z" fill="none" />
              <path d="M9 5v2h6.59L4 18.59 5.41 20 17 8.41V15h2V5H9z" />
            </svg>
          </a>
        </div>
        <span class="nav-divider" />

        <div
          v-if="languages.length > 1"
          ref="langRoot"
          class="lang-toggle"
          @mouseenter="onLangEnter"
          @mouseleave="onLangLeave"
        >
          <button
            type="button"
            class="lang-btn"
            aria-haspopup="true"
            :aria-expanded="menuOpen"
            :aria-label="(theme as any).langMenuLabel || 'Language'"
            @click.stop="toggleMenu"
          >
            <svg
              class="lang-icon"
              viewBox="0 0 24 24"
              fill="none"
              stroke="currentColor"
              stroke-width="2"
              stroke-linecap="round"
              stroke-linejoin="round"
              aria-hidden="true"
            >
              <path
                d="m5 8 6 6M4 14l6-6 2-3M2 5h12M7 2h1M22 22l-5-10-5 10M14 18h6"
              />
            </svg>
            <svg
              class="lang-chevron"
              viewBox="0 0 24 24"
              fill="none"
              stroke="currentColor"
              stroke-width="2"
              stroke-linecap="round"
              stroke-linejoin="round"
              aria-hidden="true"
            >
              <path d="m6 9 6 6 6-6" />
            </svg>
          </button>
          <div class="lang-menu" :class="{ open: menuOpen }">
            <a
              v-for="lang in languages"
              :key="lang.code"
              class="lang-option"
              :class="{ selected: lang.code === currentLang }"
              :href="pathForLang(lang.code)"
              @click.prevent="goLang(lang.code)"
            >
              <span v-if="lang.flag" class="lang-flag">{{ lang.flag }}</span>
              <span>{{ lang.label }}</span>
            </a>
          </div>
        </div>
        <span class="nav-divider" />

        <button
          type="button"
          class="appearance-switch"
          role="switch"
          :aria-checked="isDark ? 'true' : 'false'"
          :aria-label="(theme as any).darkModeSwitchLabel || 'Appearance'"
          @click="toggleAppearance"
        >
          <span class="appearance-check">
            <span class="appearance-icon">
              <svg
                class="sun"
                viewBox="0 0 24 24"
                fill="none"
                stroke="currentColor"
                stroke-width="2"
                stroke-linecap="round"
                stroke-linejoin="round"
                aria-hidden="true"
              >
                <circle cx="12" cy="12" r="4" />
                <path
                  d="M12 2v2M12 20v2M4.93 4.93l1.41 1.41M17.66 17.66l1.41 1.41M2 12h2M20 12h2M6.34 17.66l-1.41 1.41M19.07 4.93l-1.41 1.41"
                />
              </svg>
              <svg
                class="moon"
                viewBox="0 0 24 24"
                fill="none"
                stroke="currentColor"
                stroke-width="2"
                stroke-linecap="round"
                stroke-linejoin="round"
                aria-hidden="true"
              >
                <path d="M12 3a6 6 0 0 0 9 9 9 9 0 1 1-9-9Z" />
              </svg>
            </span>
          </span>
        </button>
        <span class="nav-divider" />

        <a
          v-for="(link, index) in socialLinks"
          :key="index"
          class="icon-btn"
          :href="link.link"
          target="_blank"
          rel="noopener noreferrer"
          :aria-label="link.ariaLabel || link.icon"
        >
          <VPIcon :icon="String(link.icon).includes(':') ? link.icon : `simple-icons:${link.icon}`" />
        </a>
      </div>
    </div>
  </div>
</template>
