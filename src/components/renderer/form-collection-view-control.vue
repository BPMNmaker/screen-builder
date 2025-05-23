<template>
  <vue-form-renderer
    ref="collectionViewControl"
    class="form-collection-record-control"
    :placeholder="placeholder"
    v-model="data"
    mode="preview"
    :config="validatedConfig"
    :computed="computed"
    :custom-css="customCss"
    :watchers="watchers"
    :_parent="_parent"
  />
</template>

<script>
import VueFormRenderer from "../vue-form-renderer.vue";

const globalObject = typeof window === "undefined" ? global : window;

const defaultConfig = [
  {
    name: "empty",
    items: []
  }
];

export default {
  components: {
    VueFormRenderer
  },
  props: {
    name: String,
    validationData: null,
    _parent: null,
    record: null,
    collection: {
      type: Object
    },
    taskdraft: Object,
  },
  data() {
    return {
      localData: {},
      config: defaultConfig,
      computed: [],
      customCSS: null,
      watchers: [],
      screenTitle: null,
      selCollectionId: Number,
      selRecordId: Number,
      selDisplayMode: String,
      screenCollectionId: null,
      placeholder: "Select a collection",
      screenType: "",
      hasMustache: false,
      flagDraft: {},
      taskDraft: {},
      collectionmode: "View"
    };
  },
  computed: {
    validatedConfig() {
      return this.config && this.config[0] ? this.config : defaultConfig;
    },
    data: {
      get() {
        if(this.hasMustache) {
          this.clearDataObject();
        }
        return this.localData;
      },
      set(data) {
        Object.keys(data).forEach((variable) => {
          this.validationData && this.$set(this.validationData, variable, data[variable]);
        });

        if (this.collection) {
          this.$set(this.collection, 'data', Array.isArray(data) ? data : [data]);
          this.$set(this.collection, 'screen', this.screenCollectionId);
        }
      },
    },
  },
  methods: {
    isSubmitButton(item) {
      return (
        item.config &&
        item.component === "FormButton" &&
        item.config.event === "submit"
      );
    },
    hideSubmitButtons(config) {
      config.forEach((item) => {
        //If the element has containers
        if (Array.isArray(item)) {
          this.hideSubmitButtons(item);
        }

        //If the element has items
        if (item.items) {
          this.hideSubmitButtons(item.items);
        }

        //hidden buttons
        if (this.isSubmitButton(item)) {
          item.config.hidden = true;
        }
      });
    },
    disableForm(config) {
      config.forEach((item) => {
        //If the element has containers
        if (Array.isArray(item)) {
          this.disableForm(item);
        }

        //If the element has items
        if (item.items) {
          this.disableForm(item.items);
        }

        //Disable element
        if (item && item.config) {
          item.config.disabled = true;
          item.config.readonly = true;
          item.config.editable = false;
        }
      });
    },
    loadScreen(id) {
      this.config = defaultConfig;
      this.computed = [];
      this.customCSS = null;
      this.watchers = [];
      this.screenTitle = null;

      if (id) {
        this.$dataProvider.getScreen(id).then((response) => {
          this.config = response.data.config;
          this.hideSubmitButtons(this.config);
          this.computed = response.data.computed;
          this.customCSS = response.data.custom_css;
          this.watchers = response.data.watchers;
          this.screenTitle = response.data.title;

          if (this.$attrs["disabled"]) {
            this.disableForm(this.config);
          }
        });
      }
    },
    callbackRecord() {
      this.hasMustache = true;
      this.loadRecordCollection(this.selCollectionId, 1, this.selDisplayMode);
    },
    errors() {
      this.$refs.nestedScreen.isValid();
      return this.$refs.nestedScreen.errors;
    },
    loadRecordCollection(collectionId, recordId, modeId) {
      this.selCollectionId = collectionId;
      this.selRecordId = recordId;
      this.selDisplayMode = modeId;

      this.$dataProvider
        .getCollectionRecordsView(collectionId, recordId)
        .then((response) => {
          this.placeholder = "";
          const respData = response.data;
          const viewScreen = response.collection.read_screen_id;
            //Choose screen id regarding of the display Mode
          this.screenCollectionId = viewScreen;
          this.loadScreen(this.screenCollectionId);

          //This section validates if Collection has draft data
          if(this.taskDraft?.draft?.data == null || this.taskDraft.draft.data === '') {
            this.localData = respData;
          }else{
            this.localData = this.taskDraft.draft.data;
          }
          
        })
        .catch(() => {
          this.localData = {};
          globalObject.ProcessMaker.alert(this.$t('This content does not exist. We could not locate indicated data'), "danger");
        });;
    },
    isMustache(record) {
      return /\{\{.*\}\}/.test(record);
    },
    clearDataObject() {
      Object.keys(this.localData).forEach(key => {
        if (key !== "id") {
          this.localData[key] = "";
        }
      });
    },
  },
  watch: {
    collection(collection) {
      if(collection) {
        this.selCollectionId = collection.collectionId;
        this.$root.$emit("collection-screen-mode", "display");
      }
    },
    record(record) {
      this.hasMustache = false;
      if (record && !isNaN(record) && record > 0 && this.collection.collectionId) {
        this.selRecordId = record;
        this.loadRecordCollection(this.collection.collectionId, record, this.collectionmode);
      } else {
        if (this.isMustache(record)) {
          this.callbackRecord();
        }
        this.localData = {};
      }
    }
  },
  mounted() {
    this.$root.$on("taskdraft-input", (val)=>{
      this.taskDraft = val;
    });

    if (this.collection && this.record) {
      this.loadRecordCollection(this.collection.collectionId, this.record, this.collectionmode);
    }
  },
};
</script>

<style lang="scss">
.prevent-interaction.form-collection-view-control::after {
  content: attr(placeholder);
}
</style>
