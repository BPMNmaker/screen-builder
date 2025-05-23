<template>
  <!--
    Dropdown component to display options related to pages.
    Provides options to add a new page, see all pages, and select individual pages.
  -->
  <b-dropdown
    ref="pageDropdown"
    data-test="page-dropdown"
    variant="platform"
    :boundary="boundary"
    menu-class="page-dropdown-menu"
  >
    <!-- Dropdown button content -->
    <template #button-content>
      <!-- Icon representing a file -->
      <i class="fa fa-file"></i>
    </template>

    <!-- Option to add a new page -->
    <b-dropdown-item data-test="add-page" @click="onAddPage">
      <!-- Icon for adding a new page -->
      <i class="fas fa-plus platform-dropdown-item-icon text-dark w-icon text-center"></i>
      <!-- Text for adding a new page -->
      {{ $t("Create Page") }}
    </b-dropdown-item>

    <!-- Option to see all pages -->
    <b-dropdown-item data-test="see-all-pages" @click="onSeeAllPages">
      <!-- Icon for seeing all pages -->
      <i class="far fa-eye platform-dropdown-item-icon text-dark w-icon text-center"></i>
      <!-- Text for seeing all pages -->
      {{ $t("See all pages") }}
    </b-dropdown-item>

    <!-- Option to open Clipboard page -->
    <b-dropdown-item data-test="clipboard" @click="onClipboard" class="d-flex">
      <!-- Icon for clipboard -->
      <i class="far fa-clipboard platform-dropdown-item-icon text-dark w-icon text-center"></i>
      <!-- Text for clipboard -->
      {{ $t("Clipboard") }}
    </b-dropdown-item>

    <!-- Divider between adding and viewing options -->
    <b-dropdown-divider></b-dropdown-divider>

    <!-- Dropdown items for selecting individual pages -->
    <b-dropdown-item
      v-for="(item, page) in data"
      :key="page"
      :data-test="'page-' + item.name"
      :data-cy="'page-' + page"
      @click="onClickPage(page)"
    >
      <!-- Display the name of the page -->
      {{ item.name }}
    </b-dropdown-item>
  </b-dropdown>
</template>

<script>
/**
 * Vue component for managing pages through a dropdown menu.
 * @component
 * @prop {Props} props - The component's props object.
 */

export default {
  /**
   * The name of the component.
   * @type {string}
   */
  name: "PagesDropdown",

  /**
   * The props that the component accepts.
   * @type {Object}
   * @property {PageItem[]} data - The array of page items to be displayed in the dropdown.
   *                               Defaults to null.
   */
  props: {
    data: {
      type: Array,
      default: null
    },
    boundary: {
      type: String,
      default: "viewport"
    }
  },

  /**
   * The methods available within the component.
   */
  methods: {
    /**
     * Handler for when the "Add Page" option is clicked.
     * Emits the "addPage" event.
     */
    onAddPage() {
      this.$emit("addPage");
    },

    /**
     * Handler for when the "See All Pages" option is clicked.
     * Emits the "seeAllPages" event.
     */
    onSeeAllPages() {
      this.$emit("seeAllPages");
    },

    /**
     * Handle when user clicks on "Clipboard
     */
    onClipboard() {
      this.$emit("clipboard");
    },

    /**
     * Handler for when a specific page is clicked.
     * Emits the "clickPage" event with the selected page.
     * @param {PageItem} page - The selected page item.
     */
    onClickPage(page) {
      this.$emit("clickPage", this.data[page]);
    }
  }
};
</script>

<style lang="scss" scoped>
// Platform btn style
.btn-platform {
  background-color: #ffff;
  color: #6a7888;
}
.platform-dropdown-item-icon {
  // Style for the icons in dropdown items.
  color: #1572c2;
}
.w-icon {
  width: 1.25rem;
}
</style>
