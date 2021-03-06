<template>
  <div
    ref="menuBarRef"
    :class="[dockClass, 'menu-bar-container', expandClass]"
    :draggable="draggable"
    tabindex="0"
    :style="{ background: theme.primary }"
    @dragstart="handleDragStart"
    @dragover="handleDrag"
    @dragend="handleDragEnd"
    @touchEnd="handleDragEnd"
    @mouseenter="handleMouseEnter"
    @mouseleave="handleMouseLeave"
  >
    <ul
      :class="[dockClass, 'menu-bar-items']"
      draggable="true"
      @dragstart="handleDragCancel"
    >
      <li
        v-for="item of menuItems"
        :key="item.id"
        :class="[dockClass, 'v-dock-menu-bar-item-wrapper']"
      >
        <menu-bar-item
          :id="item.id"
          :dock="dockPosition"
          :menu-active="menuActive"
          :menu-bar-dimensions="{ height: barHeight, width: barWidth }"
          :menu="item.menu"
          :name="item.name"
          :menu-bar-active="menuBarActive"
          :show-menu="item.showMenu"
          :theme="theme"
          :is-touch-device="isMobileDevice"
          :on-selected="handleSelected"
          :highlight-first-element="highlightFirstElement"
          @activate="handleActivateMenu"
          @activate-next="handleActivateDir"
          @activate-previous="handleActivateDir"
          @show="handleOnShowMenu"
        >
          <template
            v-for="slot in Object.keys($slots)"
            #[slot]="scope"
          >
            <slot
              :name="slot"
              v-bind="scope"
            />
          </template>
        </menu-bar-item>
      </li>
    </ul>
  </div>
</template>

<script lang="ts">
import {
  computed,
  defineComponent,
  PropType,
  ref,
  onMounted,
  unref,
  onUnmounted,
} from "vue";
import MenuBarItem from "./MenuBarItem.vue";
import DockPosition from "../models/MenuBarDockPosition";
import { MenuBarItemModel } from "@/models/MenuBarItemModel";
import "focus-visible";
import isMobile from "./isMobileDevice";
import utils from "../utils/DragUtil";
import { MenuTheme } from "@/models/Theme";

export default defineComponent({
  name: "MenuBar",
  components: {
    MenuBarItem,
  },
  props: {
    items: {
      required: true,
      default: [] as MenuBarItemModel[],
      type: Array as PropType<MenuBarItemModel[]>,
    },
    dock: {
      required: false,
      default: DockPosition.TOP,
      type: String,
    },
    onSelected: {
      required: true,
      type: Function as PropType<
        ({ name, path }: { name: string; path: string }) => void
      >,
    },
    draggable: {
      required: false,
      type: Boolean,
      default: true,
    },
    theme: {
      required: false,
      type: Object as PropType<MenuTheme>,
      default: () => ({
        primary: "#21252b",
        secondary: "#32323e",
        tertiary: "#4c4c57",
        textColor: "#fff",
      }),
    },
  },
  setup(props) {
    // reference to the main menubar itself
    const menuBarRef = ref<HTMLElement>();

    // tracks the drag status of the menubar
    const dragActive = ref(false);

    // tracks if the menu is active
    const menuActive = ref(false);

    // tracks if the menubar is active
    const menuBarActive = ref(false);

    // reference to the dock position
    const dockPosition = ref<string>(props.dock);

    // compute the dock class
    const dockClass = computed(() => [dockPosition.value.toLowerCase()]);

    // height and width of the menubar
    const barHeight = ref<number>(0);
    const barWidth = ref<number>(0);

    const isMobileDevice = ref<boolean>();

    // highlights the first element in the menu (on keyboard interaction)
    const highlightFirstElement = ref<boolean>();

    // initialize the menu items.add a unique id to all items.
    const menuItems = ref<MenuBarItemModel[]>(
      props.items.map((item) =>
        Object.assign({}, item, {
          id: Math.random().toString(16).slice(2),
        })
      )
    );

    const activeMenuSelection = ref(-1);

    // tracks the active menubar item
    const activeMenuBarId = ref("");

    const clientCoords = ref<{ x: number; y: number }>({ x: 0, y: 0 });

    const handleMenuClosure = () => {
      menuActive.value = false;
      menuBarActive.value = false;
      activeMenuSelection.value = -1;
      activeMenuBarId.value = "";
      highlightFirstElement.value = false;
    };

    //** Lifecylce  Methods **

    const updateDragCoords = (event: DragEvent) =>
      (clientCoords.value = { x: event.clientX, y: event.clientY });

    onMounted(() => {
      // get reference to the menubar
      const menu = menuBarRef.value;

      // save the bar's height and width on render
      barHeight.value = menu?.clientHeight as number;
      barWidth.value = menu?.clientWidth as number;

      // check if its a mobile device
      isMobileDevice.value = isMobile();

      document.addEventListener("dragover", updateDragCoords);
      document.addEventListener("click", handleMenuClosure);
    });

    // cleanup
    onUnmounted(() => {
      document.removeEventListener("dragover", updateDragCoords);
      document.removeEventListener("click", handleMenuClosure);
    });

    const handleDragStart = (event: DragEvent | TouchEvent) => {
      dragActive.value = true;

      // close the menu during drag operation
      menuActive.value = false;

      // set a custom ghost image while dragging
      utils.handleDragStart(event);
    };

    //** Drag handlers **

    const handleDragEnd = (event: DragEvent | TouchEvent) => {
      const {
        dragActive: dragActiveNew,
        dockPosition: positionNew,
      } = utils.handleDragEnd(event, unref(clientCoords));

      dragActive.value = dragActiveNew;
      dockPosition.value = positionNew;
    };

    const handleDrag = (event: DragEvent) => {
      clientCoords.value = {
        x: event.clientX,
        y: event.clientY,
      };
    };

    // cancels the drag event on the child elements of the menubar
    const handleDragCancel = (event: DragEvent) => {
      event.preventDefault();
      event.stopPropagation();
    };

    //** Other handlers */

    const handleActivateMenu = (id?: string) => {
      menuItems.value = menuItems.value.map((item) =>
        Object.assign({}, item, {
          showMenu: item.id === id,
        })
      );
    };

    const handleOnShowMenu = (state: boolean, id: string) => {
      menuActive.value = state;
      if (state) {
        activeMenuBarId.value = id;
      } else {
        activeMenuBarId.value = "";
        highlightFirstElement.value = false;
      }
    };

    const handleBlur = () => {
      menuActive.value = false;
      menuBarActive.value = false;
      highlightFirstElement.value = false;

      menuItems.value = menuItems.value.map((item) =>
        Object.assign({}, item, {
          showMenu: false,
        })
      );
    };

    //** computed methods */

    // convert to a sidebar when docked to either left or right
    const expandClass = computed(() => {
      if (
        dockPosition.value === DockPosition.LEFT ||
        dockPosition.value === DockPosition.RIGHT
      ) {
        return menuBarActive.value ? "expanded" : "not-expanded";
      }
    });

    //** final selection handler */
    const handleSelected = (data: any) => {
      menuActive.value = false;
      props.onSelected(data);
    };

    //** keyboard handlers  **

    // activate the menu bar
    const handleMouseEnter = () => {
      if (!isMobileDevice.value) {
        menuBarActive.value = true;
      }
    };

    // deactivate the menubar if active
    const handleMouseLeave = () => {
      if (!isMobileDevice.value && !menuActive.value) {
        menuBarActive.value = false;
      }
    };

    // activates the menu via keyboard
    const handleActivateDir = (id: string, dir: "prev" | "next") => {
      const eleIndex = menuItems.value.findIndex((item) => item.id === id);
      const newIdx = dir === "next" ? eleIndex + 1 : eleIndex - 1;
      const menuItemsLen = menuItems.value.length;

      let nextId = "";

      if (newIdx > -1 && newIdx < menuItemsLen) {
        nextId = menuItems.value[newIdx].id as string;
      } else if (newIdx > menuItemsLen - 1) {
        nextId = menuItems.value[0].id as string;
      } else if (newIdx < 0) {
        nextId = menuItems.value[menuItemsLen - 1].id as string;
      }

      // get the menubar item
      const menuBarItem = menuItems.value.find((item) => item.id === id);

      const menuItem =
        menuBarItem && menuBarItem.menu
          ? menuBarItem.menu[activeMenuSelection.value]
          : null;

      if (menuItem?.menu && dir === "next") {
        menuItems.value = menuItems.value.map((item) => {
          if (item.id === activeMenuBarId.value) {
            return Object.assign({}, item, {
              menu: item.menu?.map((it) =>
                Object.assign({}, it, {
                  showSubMenu:
                    it.name?.toLowerCase() === menuItem.name?.toLowerCase(),
                })
              ),
            });
          } else {
            return item;
          }
        });
      } else {
        // move to the next menu bar item
        highlightFirstElement.value = true;
        activeMenuBarId.value = nextId;
        nextId && handleActivateMenu(nextId);
      }

      // reset active menu selection
      activeMenuSelection.value = -1;
    };

    return {
      barHeight,
      barWidth,
      dockClass,
      dockPosition,
      handleDragEnd,
      handleDragStart,
      handleOnShowMenu,
      handleActivateMenu,
      handleMouseEnter,
      handleMouseLeave,
      handleBlur,
      menuActive,
      menuBarRef,
      menuItems,
      handleDragCancel,
      expandClass,
      menuBarActive,
      isMobileDevice,
      handleDrag,
      handleActivateDir,
      activeMenuSelection,
      activeMenuBarId,
      handleSelected,
      highlightFirstElement,
    };
  },
});
</script>

<style lang="scss" src="./menu-bar.scss" scoped>
</style>