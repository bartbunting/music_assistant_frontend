<template>
  <v-toolbar :color="color" class="header">
    <span class="sr-only" role="status" aria-live="assertive">
      {{ overflowMenuAnnouncement }}
    </span>
    <template v-if="icon" #prepend>
      <v-btn
        :icon="typeof icon === 'string' ? icon : undefined"
        size="small"
        :disabled="iconAction == null"
        :aria-label="toolbarIconLabel"
        :aria-hidden="toolbarIconLabel ? undefined : true"
        style="opacity: 0.8"
        @click="iconAction?.()"
      >
        <component :is="icon" v-if="typeof icon !== 'string'" class="w-6 h-6" />
      </v-btn>
    </template>

    <template #title>
      <slot name="title">
        <button
          v-if="title || (store.mobileLayout && isDiscoverPage)"
          @click="emit('titleClicked')"
        >
          {{ title || (isDiscoverPage ? $t("discover") : "") }}
        </button>
      </slot>
    </template>

    <template v-if="$slots.append || menuItems?.length" #append>
      <slot name="append"></slot>
      <v-btn
        v-for="menuItem of menuItems?.filter(
          (x) =>
            !x.hide &&
            !enforceOverflowMenu &&
            (getBreakpointValue('bp8') || x.overflowAllowed === false),
        )"
        :key="menuItem.label"
        variant="text"
        style="width: 40px"
        :title="$t(menuItem.label, menuItem.labelArgs || [])"
        :aria-label="$t(menuItem.label, menuItem.labelArgs || [])"
        :aria-haspopup="menuItem.subItems?.length ? 'menu' : undefined"
        :aria-pressed="menuItem.active == null ? undefined : menuItem.active"
        :disabled="menuItem.disabled == true"
        @click="(e: MouseEvent) => onMenuItemClick(e, menuItem)"
      >
        <v-badge :model-value="menuItem.active == true" color="primary" dot>
          <v-icon
            v-if="typeof menuItem.icon === 'string'"
            :icon="menuItem.icon"
            :color="$vuetify.theme.current.dark ? '#fff' : '#000'"
            size="22px"
          />
          <component
            :is="menuItem.icon"
            v-else-if="menuItem.icon"
            class="w-[22px] h-[22px]"
            :color="$vuetify.theme.current.dark ? '#fff' : '#000'"
          />
        </v-badge>
      </v-btn>

      <!-- overflow menu with (remaining) items if on mobile -->
      <div
        v-if="
          (!getBreakpointValue('bp8') || enforceOverflowMenu) &&
          menuItems?.filter(
            (x) => x.hide != true && x.overflowAllowed !== false,
          ).length
        "
      >
        <v-menu
          v-model="overflowMenuOpen"
          location="bottom end"
          attach=".v-application"
          content-class="voiceover-options-menu"
          :content-props="optionsMenuContentProps"
          :close-on-content-click="false"
          eager
        >
          <template #activator="{ props }">
            <v-btn
              variant="plain"
              style="width: 15px; margin-left: -10px"
              v-bind="props"
              :aria-label="overflowMenuButtonLabel"
              :aria-controls="overflowMenuContentId"
              aria-haspopup="menu"
              :aria-expanded="overflowMenuOpen ? 'true' : 'false'"
              @click="rememberOverflowMenuActivator"
            >
              <v-icon
                icon="mdi-dots-vertical"
                :color="$vuetify.theme.current.dark ? '#fff' : '#000'"
                size="22"
                style="margin-right: -5px; width: 15px"
              />
            </v-btn>
          </template>
          <div
            ref="overflowMenuContentRef"
            class="options-menu-panel"
            tabindex="-1"
            @keydown.esc.stop.prevent="overflowMenuOpen = false"
          >
            <span :id="overflowMenuDebugId" class="sr-only">
              {{ overflowMenuDebugText }}
            </span>
            <v-list
              density="compact"
              slim
              tile
              role="presentation"
              tabindex="-1"
            >
              <v-list-item
                v-for="(menuItem, index) in menuItems?.filter(
                  (x) => x.hide != true && x.overflowAllowed != false,
                )"
                :key="index"
                tag="button"
                type="button"
                class="toolbar-menu-button"
                data-menu-action
                role="menuitem"
                :tabindex="menuItem.disabled == true ? -1 : 0"
                :title="$t(menuItem.label, menuItem.labelArgs || [])"
                :disabled="menuItem.disabled == true"
                :aria-disabled="menuItem.disabled == true ? 'true' : undefined"
                :aria-haspopup="menuItem.subItems?.length ? 'menu' : undefined"
                :append-icon="
                  menuItem.subItems?.length ? 'mdi-chevron-right' : undefined
                "
                @click.prevent.stop="
                  (e: MouseEvent | KeyboardEvent) =>
                    onMenuItemClick(e, menuItem)
                "
                @keydown.enter.prevent.stop="
                  (e: KeyboardEvent) => onMenuItemClick(e, menuItem)
                "
                @keydown.space.prevent.stop="
                  (e: KeyboardEvent) => onMenuItemClick(e, menuItem)
                "
              >
                <template v-if="menuItem.icon" #prepend>
                  <v-badge
                    :model-value="menuItem.active == true"
                    color="primary"
                    dot
                  >
                    <v-icon
                      v-if="typeof menuItem.icon === 'string'"
                      :icon="menuItem.icon"
                      :color="$vuetify.theme.current.dark ? '#fff' : '#000'"
                      size="22px"
                    />
                    <component
                      :is="menuItem.icon"
                      v-else
                      class="w-[22px] h-[22px]"
                      :color="$vuetify.theme.current.dark ? '#fff' : '#000'"
                    />
                  </v-badge>
                </template>
              </v-list-item>
            </v-list>
          </div>
        </v-menu>
      </div>
    </template>
  </v-toolbar>
</template>

<script setup lang="ts">
import { ContextMenuItem } from "@/layouts/default/ItemContextMenu.vue";
import { api } from "@/plugins/api";
import { eventbus } from "@/plugins/eventbus";
import { store } from "@/plugins/store";
import { getBreakpointValue } from "../plugins/breakpoint";

import type { Component } from "vue";
import { computed, nextTick, ref, useId, watch } from "vue";
import { useI18n } from "vue-i18n";

const overflowMenuOpen = ref(false);
const overflowMenuContentRef = ref<HTMLElement | null>(null);
const overflowMenuActivator = ref<HTMLElement | null>(null);
const overflowMenuAnnouncement = ref("");
const { t } = useI18n();
const overflowMenuBaseId = useId();
const overflowMenuContentId = `toolbar-overflow-menu-${overflowMenuBaseId}`;
const overflowMenuDebugId = `toolbar-overflow-menu-debug-${overflowMenuBaseId}`;
const overflowMenuDebugText = "MA toolbar options menu test";
const overflowMenuButtonLabel = computed(
  () => `${t("more_options")} ${overflowMenuDebugText}`,
);
const optionsMenuContentProps = computed(() => ({
  id: overflowMenuContentId,
  role: "menu",
  "aria-label": `${t("more_options")} ${overflowMenuDebugText}`,
  "aria-describedby": overflowMenuDebugId,
  "data-vo-debug-marker": "ma-toolbar-options-menu-2026-05-14",
}));

const focusOverflowMenu = (attempts = 5) => {
  nextTick(() => {
    requestAnimationFrame(() => {
      const menuContent = overflowMenuContentRef.value;
      if (!menuContent) {
        if (attempts > 0) {
          window.setTimeout(() => focusOverflowMenu(attempts - 1), 50);
        }
        return;
      }
      const firstMenuItem = menuContent.querySelector<HTMLElement>(
        "[data-menu-action]:not([aria-disabled='true'])",
      );
      (firstMenuItem || menuContent).focus({ preventScroll: true });

      if (attempts > 0) {
        window.setTimeout(() => {
          if (
            overflowMenuOpen.value &&
            !menuContent.contains(document.activeElement)
          ) {
            focusOverflowMenu(attempts - 1);
          }
        }, 50);
      }
    });
  });
};

const rememberOverflowMenuActivator = () => {
  if (document.activeElement instanceof HTMLElement) {
    overflowMenuActivator.value = document.activeElement;
  }
};

const announceOverflowMenuOpen = () => {
  overflowMenuAnnouncement.value = "";
  nextTick(() => {
    if (overflowMenuOpen.value) {
      overflowMenuAnnouncement.value = overflowMenuDebugText;
    }
  });
};

watch(overflowMenuOpen, (open) => {
  if (open) {
    announceOverflowMenuOpen();
    focusOverflowMenu();
    return;
  }

  overflowMenuAnnouncement.value = "";
  if (overflowMenuActivator.value?.isConnected) {
    overflowMenuActivator.value.focus({ preventScroll: true });
  }
  overflowMenuActivator.value = null;
});

const onMenuItemClick = (
  event: MouseEvent | KeyboardEvent,
  menuItem: ToolBarMenuItem,
) => {
  event.preventDefault();
  if (menuItem.disabled) return;

  if (menuItem.subItems?.length) {
    // Open submenu via global context menu
    // Map closeOnContentClick to close_on_click on subItems if needed
    const items =
      menuItem.closeOnContentClick === false
        ? menuItem.subItems.map((item) => ({
            ...item,
            close_on_click: item.close_on_click ?? false,
          }))
        : menuItem.subItems;
    const posX = "clientX" in event ? event.clientX : 0;
    const posY = "clientY" in event ? event.clientY : 0;
    eventbus.emit("contextmenu", {
      items,
      posX,
      posY,
    });
  } else if (menuItem.action) {
    // Close overflow menu before executing action
    overflowMenuOpen.value = false;
    // Execute direct action
    menuItem.action();
  }
};

// properties
interface Props {
  color?: string;
  icon?: string | Component;
  title?: string;
  menuItems?: ToolBarMenuItem[];
  enforceOverflowMenu?: boolean;
  isDiscoverPage?: boolean;
  iconAction?: () => void;
  iconLabel?: string;
}
const toolbarProps = withDefaults(defineProps<Props>(), {
  color: "transparent",
  icon: undefined,
  title: undefined,
  count: undefined,
  menuItems: undefined,
  enforceOverflowMenu: false,
  iconAction: undefined,
  iconLabel: undefined,
});

const toolbarIconLabel = computed(() => {
  if (toolbarProps.iconLabel) return toolbarProps.iconLabel;
  if (toolbarProps.title) return toolbarProps.title;
  if (toolbarProps.isDiscoverPage) return t("discover");
  if (toolbarProps.iconAction) return t("back");
  return undefined;
});

// emitters
const emit = defineEmits<{
  (e: "iconClicked"): void;
  (e: "titleClicked"): void;
}>();
</script>

<script lang="ts">
export interface ToolBarMenuItem extends ContextMenuItem {
  active?: boolean;
  subItems?: ContextMenuItem[];
  overflowAllowed?: boolean;
  closeOnContentClick?: boolean;
}
</script>

<style scoped>
.header.v-toolbar {
  height: 55px;
  font-family: "JetBrains Mono Medium";
}

.header.v-toolbar :deep(.v-toolbar__content) {
  height: 55px !important;
  min-height: 55px !important;
  padding-top: 0;
  padding-bottom: 0;
  align-items: center;
}

.header.v-toolbar :deep(.v-toolbar-title) {
  margin-inline-start: 10px !important;
}

.header.v-toolbar :deep(.v-toolbar__prepend) {
  margin-inline-start: 12px !important;
  margin-inline-end: 0px !important;
}

.header.v-toolbar > .v-toolbar__content > .v-toolbar__append {
  margin-inline-end: 5px;
}

.header.v-toolbar-default > .v-toolbar__content > .v-toolbar__append {
  margin-inline-end: 10px;
}

:global(.voiceover-options-menu) {
  contain: none !important;
}

.toolbar-menu-button {
  width: 100%;
  border: 0;
  text-align: start;
}

/* Mobile branding on the left */
.toolbar-prepend {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}
</style>
