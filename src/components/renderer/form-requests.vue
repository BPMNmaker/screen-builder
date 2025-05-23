<template>
  <div v-if="showTable">
    <filter-table
      table-name="form-cases"
      :headers="tableHeaders"
      :data="tableData"
      :unread="unreadColumnName"
      :loading="shouldShowLoader"
    >
      <template v-for="(row, rowIndex) in data.data" #[`row-${rowIndex}`]>
        <td
          v-for="(header, colIndex) in tableHeaders"
          :key="`${rowIndex}-${colIndex}`"
          :class="{
            'pm-table-filter-applied-tbody': header.sortAsc || header.sortDesc
          }"
        >
          <template v-if="containsHTML(getNestedPropertyValue(row, header))">
            <div
              :id="`element-${rowIndex}-${colIndex}`"
              :class="{ 'pm-table-truncate': header.truncate }"
              :style="{ maxWidth: header.width + 'px' }"
            >
              <span v-html="sanitize(getNestedPropertyValue(row, header))"></span>
            </div>
            <b-tooltip
              v-if="header.truncate"
              :target="`element-${rowIndex}-${colIndex}`"
              custom-class="pm-table-tooltip"
              placement="topright"
              trigger="hover"
              boundary="viewport"
              :delay="{ show: 0, hide: 0 }"
              @show="checkIfTooltipIsNeeded"
            >
              {{ sanitizeTooltip(getNestedPropertyValue(row, header)) }}
            </b-tooltip>
          </template>
          <template v-else>
            <template v-if="isComponent(row[header.field])">
              <component
                :is="row[header.field].component"
                v-bind="row[header.field].props"
              />
            </template>
            <template v-else>
              <div
                :id="`element-${rowIndex}-${colIndex}`"
                :class="{ 'pm-table-truncate': header.truncate }"
                :style="{ maxWidth: header.width + 'px' }"
              >
                {{ getNestedPropertyValue(row, header) }}
                <b-tooltip
                  v-if="header.truncate"
                  :target="`element-${rowIndex}-${colIndex}`"
                  custom-class="pm-table-tooltip"
                  placement="topright"
                  trigger="hover"
                  boundary="viewport"
                  :delay="{ show: 0, hide: 0 }"
                  @show="checkIfTooltipIsNeeded"
                >
                  {{ getNestedPropertyValue(row, header) }}
                </b-tooltip>
              </div>
            </template>
          </template>
        </td>
      </template>
    </filter-table>
  </div>
  <div v-else>
    <formEmpty link="Requests" :title="$t('No Cases to Show')" :url="noDataUrl" />
  </div>
</template>

<script>
import { createUniqIdsMixin } from "vue-uniq-ids";
import datatableMixin from "../../mixins/datatable";
import formEmpty from "./form-empty-table.vue";

const uniqIdsMixin = createUniqIdsMixin();
export default {
  components: { formEmpty },
  mixins: [uniqIdsMixin, datatableMixin],
  data() {
    return {
      countResponse: "0",
      fields: [],
      filter: "",
      data: [],
      tableData: [],
      orderBy: "id",
      orderDirection: "DESC",
      additionalParams: "",
      showTable: true,
      pmqlSearch: "",
      sortOrder: [
        {
          field: "id",
          sortField: "id",
          direction: "desc"
        }
      ],
      tableHeaders: []
    };
  },
  computed: {
    noDataUrl() {
      return `${window.ProcessMaker?.app?.url}/cases`;
    }
  },
  mounted() {
    this.setupColumns();
    this.pmql = `requester = "${Processmaker.user.username}"`;
    this.fetch();
    this.$root.$on("dropdownSelectionRequest", this.fetchData);
    this.$root.$on("searchRequest", this.fetchSearch);
  },
  methods: {
    fetch() {
      Vue.nextTick(() => {
        let pmql = "";

        if (this.pmql !== undefined) {
          pmql = this.pmql;
        }

        if (this.pmqlSearch) {
          pmql = pmql + "AND" + this.pmqlSearch;
        }

        let { filter } = this;

        if (filter && filter.length) {
          if (filter.isPMQL()) {
            pmql = `(${pmql}) and (${filter})`;
            filter = "";
          }
        }

        if (this.previousFilter !== filter) {
          this.page = 1;
        }

        this.previousFilter = filter;

        if (this.previousPmql !== pmql) {
          this.page = 1;
        }

        this.previousPmql = pmql;

        // Load from our api client
        ProcessMaker.apiClient
          .get(
            `requests?page=${this.page}&include=process,participants,data` +
              `&pmql=${encodeURIComponent(pmql)}
              &per_page=${this.perPage}&filter=${filter}&order_by=${
                this.orderBy === "__slot:ids" ? "id" : this.orderBy
              }&order_direction=${this.orderDirection}${this.additionalParams}`
          )
          .then((response) => {
            this.showTable = response.data.data.length !== 0;
            for (const record of response.data.data) {
              record.case_number = this.formatOpenCase(record, "case_number");
              record.case_title = this.formatOpenCase(record, "case_title");
              record.status = this.formatStatus(record.status);
            }
            this.tableData = response.data;
            this.countResponse = this.tableData.meta.total;
            const dataControls = {
              count: `${this.countResponse}`,
              showControl: true,
              showAvatar: true,
              variant: "primary",
              textColor: "text-primary",
              colorText: "color: #1572C2",
              url: "/cases",
              dropdownShow: "requests"
            };
            const tasksDropdown = [];
            this.$emit("requestsCount", { dataControls, tasksDropdown });
          })
          .catch(() => {
            this.tableData = [];
          });
      });
    },
    formatStatus(status) {
      let color = "";
      let label = "";
      switch (status) {
        case "DRAFT":
          color = "danger";
          label = "Draft";
          break;
        case "CANCELED":
          color = "danger";
          label = "Canceled";
          break;
        case "COMPLETED":
          color = "primary";
          label = "Completed";
          break;
        case "ERROR":
          color = "danger";
          label = "Error";
          break;
        default:
          color = "success";
          label = "In Progress";
      }
      return `<span class="badge badge-${color} status-${color}"> ${label} </span>`;
    },
    /**
     * Add the formart to column to open a case in other tab
     */
    formatOpenCase(value, option) {
      const attr = value;
      if (option === "case_title") {
        attr[option] = value.case_title_formatted || value.case_title || "";
      }
      return `
        <a href="${this.openRequest(value)}" class="text-nowrap custom-wrap"
          target="_blank">${attr[option]}</a>`;
    },
    openRequest(data) {
      return `/requests/${data.id}`;
    },
    classDueDate(value) {
      const dueDate = moment(value);
      const now = moment();
      const diff = dueDate.diff(now, "hours");
      return diff < 0
        ? "text-danger"
        : diff <= 1
        ? "text-warning"
        : "text-dark";
    },
    fetchData(selectedOptions) {
      if (selectedOptions[0] === "by_me" && selectedOptions[1] !== "View All") {
        this.pmql = `(user_id = ${ProcessMaker.user.id}) AND (status = "${selectedOptions[1]}")`;
      }
      if (
        selectedOptions[0] === "as_participant" &&
        selectedOptions[1] !== "View All"
      ) {
        this.pmql = `(status = "${selectedOptions[1]}") AND (participant = "${Processmaker.user.username}")`;
      }
      if (selectedOptions[1] === "View All") {
        this.pmql = `(user_id = ${ProcessMaker.user.id}) AND ((status = "In Progress") OR (status = "Completed"))`;
      }
      this.fetch();
    },
    fetchSearch(searchData) {
      this.pmqlSearch = "";
      this.pmqlSearch = searchData;
      this.fetch();
    },
    setupColumns() {
      const columnsCases = this.getColumnsCases();
      this.tableHeaders = this.getColumnsCases();

      columnsCases.forEach((column) => {
        const field = {
          title: () => this.$t(column.label)
        };

        switch (column.field) {
          case "case_number":
            field.name = "__slot:case_number";
            break;
          case "case_title":
            field.name = "__slot:case_title";
            break;
          default:
            field.name = column.name || column.field;
        }

        if (!field.field) {
          field.field = column.field;
        }

        if (column.format === "datetime") {
          field.callback = "formatDateUser|datetime";
        }

        if (column.format === "date") {
          field.callback = "formatDateUser|date";
        }

        if (column.sortable === true && !field.sortField) {
          field.sortField = column.field;
        }

        this.fields.push(field);
      });
    },
    getColumnsCases() {
      return [
        {
          label: "Case #",
          field: "case_number",
          sortable: true,
          default: true,
          width: 96,
          fixed_width: 96
        },
        {
          label: "Case title",
          field: "case_title",
          sortable: true,
          default: true,
          truncate: true,
          width: 314,
          fixed_width: 314
        },
        {
          label: "Status",
          field: "status",
          sortable: true,
          default: true,
          width: 113,
          fixed_width: 113,
          filter_subject: { type: "Status" }
        },
        {
          label: "",
          field: "options",
          sortable: false,
          width: 10
        }
      ];
    }
  }
};
</script>

<style scoped>
.pm-table-container {
  height: 300px;
}
</style>
