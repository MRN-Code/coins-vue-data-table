<template>
  <div class="coins-data-table-container">
    <div v-show="!loading" class="coins-data-table-controls">
      <button v-if="exportButton" @click="exportCSV">Export CSV</button>

      <button v-if="columnsButton" @click="columnFilter = !columnFilter">Columns</button>
      <transition name="fade">
        <div class="coins-data-table-column-filter" v-show="columnFilter">
          <span v-for="column in columns">
            <label>
              <input type="checkbox" :value="column.name" v-model="visibleColumnNames"></input>
              {{ column.name }}
            </label>
          </span>
        </div>
      </transition>

      <button v-for="button in buttons" @click="button.click(getCheckedRows())">{{ button.title }}</button>

      <label v-if="filter" class="coins-data-table-filter">
        Filter:&nbsp
        <input v-model="filterText"></input>
      </label>
    </div>

    <table id="coins-data-table">
      <thead>
        <tr>
          <th v-if="checkboxes" class="a-checkbox">
            <label><input type="checkbox" v-model="allChecked"></label>
          </th>
          <th v-for="column in visibleColumns" @click="updateSort(column)">
            <span class="header-title">{{ column.name }}</span>
            <span v-if="column.type !== 'button'" class="coins-data-table-sort-icon"
                 :class="{ asc: column.sort === 'asc', desc: column.sort === 'desc' }">
            </span>
          </th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="row in displayedRows">
          <td v-if="checkboxes" class="a-checkbox">
            <label><input type="checkbox" v-model="row.checked"></label>
          </td>
          <td v-for="column in visibleColumns">
            <span v-if="column.type === 'button'">
              <div class="pad">
                <span class="button" :class="column.data.class(row)" @click="column.data.click(row)">{{ column.data.title }}</span>
              </div>
            </span>
            <span v-else-if="column.type === 'html'" v-html="renderCell(row, column.data)"></span>
            <span v-else>
              {{ renderCell(row, column.data) }}
            </span>
            <span v-if="column.tooltip" class="tooltip-text">
              {{ renderCell(row, column.tooltip) }}
            </span>
          </td>
        </tr>
      </tbody>
    </table>

    <div v-if="loading" class="spinner"></div>

    <div v-show="!loading" class="coins-data-table-page-select">
      <div class="coins-data-table-pagination-controls left">
        <button v-if="page > 1" @click="page = Math.max(page - skipRange, 1)">&lt;&lt;</button>
        <button v-else class="disabled">&lt;&lt;</button>

        <button v-if="page > 1" @click="page = page - 1">&lt;</button>
        <button v-else class="disabled">&lt;</button>
      </div>

      <button v-for="index in paginationDetails.visibleIndicies"
            @click="page = index"
            :class="{ current: page === index }">
        {{ index }}
      </button>

      <div class="coins-data-table-pagination-controls right">
        <button v-if="page < pages" @click="page = page + 1">&gt;</button>
        <button v-else class="disabled">&gt;&gt;</button>

        <button v-if="page < pages" @click="page = Math.min(page + skipRange, pages)">&gt;&gt;</button>
        <button v-else class="disabled">&gt;</button>
      </div>
    </div>
  </div>
</template>

<script>
    export default {
        props: {
            dataSource: {
                type: [String, Array],
                required: true
            },
            columns: {
                type: Array,
                required: true
            },
            checkboxes: {
                type: Boolean,
                default: false
            },
            secondarySort: {
                type: [ Object, Boolean ],
                default: false
            },

            // Pagination
            rowsPerPage: {
                type: Number,
                default: 20
            },
            visiblePageRange: {
                type: Number,
                default: 6
            },
            skipRange: {
                type: Number,
                default: 5
            },

            // Control Toggles
            exportButton: {
                type: Boolean,
                default: true
            },
            columnsButton: {
                type: Boolean,
                default: true
            },
            buttons: {
                type: Array
            },
            filter: {
                type: Boolean,
                default: true
            }
        },

        data() {
            return {
                data: [],
                filterText: '',
                page: 1,
                loading: true,
                sort: null,
                columnFilter: false,
                visibleColumnNames: [],
                allChecked: false
            }
        },

        computed: {
            // Filter data if filter text is present
            filteredRows() {
                let filtered = [];

                // Filter box
                if (this.filterText === '') {
                    filtered = this.data;
                } else {
                    filtered = this.data.filter((row) => {
                        return this.columns.some((column) => {
                            if (column.type !== 'button' && this.renderCell(row, column.data)) {
                                return this.renderCell(row, column.data).toLowerCase()
                                    .indexOf(this.filterText.toLowerCase()) !== -1;
                            } else {
                                return false;
                            }
                        });
                    });
                }

                // Column sort
                if (this.sort) {
                    return filtered.sort((rowA, rowB) => {
                        let a = this.renderCell(rowA, this.sort.name) || '';
                        let b = this.renderCell(rowB, this.sort.name) || '';
                        let sortOverride = false;

                        if (a === b && this.secondarySort) {
                            a = this.renderCell(rowA, this.secondarySort.name) || '';
                            b = this.renderCell(rowB, this.secondarySort.name) || '';
                            sortOverride = this.secondarySort.type;
                        }

                        if ((sortOverride && sortOverride === 'numeric') || this.sort.type === 'numeric') {
                            if (this.sort.order === 'asc') {
                                return a - b;
                            } else {
                                return b - a;
                            }
                        } else {
                            if (this.sort.order === 'asc') {
                                return a.localeCompare(b);
                            } else {
                                return b.localeCompare(a);
                            }
                        }
                    });
                } else {
                    return filtered;
                }
            },
            pages() {
                return Math.ceil(this.filteredRows.length / this.rowsPerPage);
            },
            displayedRows() {
                return this.filteredRows.slice((this.page - 1) * this.rowsPerPage, this.page * this.rowsPerPage);
            },
            paginationDetails() {
                const details = {};

                let range = [ this.page ];
                while (range.length <= this.visiblePageRange && range.length < this.pages) {
                    if (range[0] > 1) {
                        range.unshift(range[0] - 1);
                    }

                    if (range[range.length - 1] < this.pages) {
                        range.push(range[range.length - 1] + 1);
                    }
                }

                details.visibleIndicies = range;

                return details;
            },
            visibleColumns() {
                return this.columns.filter((column) => {
                    return this.visibleColumnNames.indexOf(column.name) !== -1;
                });
            }
        },

        watch: {
            pages(pages) {
                this.page = Math.max(Math.min(this.page, pages), 1);
            },
            dataSource(dataSource) {
                this.getTableData();

                this.columns.forEach((column) => {
                    this.visibleColumnNames.push(column.name);
                });
            },
            allChecked(allChecked) {
                this.data.forEach((row) => {
                    row.checked = allChecked;
                });
            }
        },

        mounted() {
            this.columns.forEach((column) => {
                this.visibleColumnNames.push(column.name);
            });

            this.getTableData();
        },

        methods: {
            getTableData() {
                this.loading = true;

                this.columns.forEach((column) => {
                    if (column.visible !== false) {
                        this.visibleColumnNames.push(column.name);
                    }
                });

                if (this.dataSource instanceof Array) {
                    this.data = this.dataSource;
                    this.loading = false;
                    return;
                }

                // URL (string) given
                axios.get(this.dataSource)
                    .then(({ data: { data } }) => { this.data = data; this.loading = false; })
                    .catch(() => { alert('There was an error getting the table data.'); });
            },

            renderCell(row, data) {
                if (!data) return '';

                // Prioritize function types
                if (typeof data === 'function') {
                    return data(row).toString();
                }

                // Parse nested properties
                const keys = data.split('.');
                data = row[keys[0]];

                for (let i = 1; i < keys.length; i++) {
                    data = data[keys[i]];
                }

                // Booleans default to true -> Yes / false-> No
                if (typeof data === 'boolean') {
                    return data ? 'Yes' : 'No';
                }

                if (typeof data === 'array') {
                    return data.join(', ');
                }

                if (!data) {
                    return '';
                }

                return data.toString();
            },
            updateSort(column) {
                if (column.type === 'button') return;

                this.columns.forEach((aColumn) => {
                    if (aColumn === column) {
                        if (!aColumn.sort || aColumn.sort === 'desc') {
                            aColumn.sort = 'asc';
                        } else {
                            aColumn.sort = 'desc';
                        }
                    } else {
                        aColumn.sort = null;
                    }
                });

                if (column.sort) {
                    this.sort = { name: column.data, order: column.sort, type: column.type };
                } else {
                    this.sort = null;
                }
            },
            getCheckedRows() {
                if (!this.checkboxes) return this.data;

                return this.data.filter((row) => {
                    return row.checked;
                });
            },

            exportCSV() {
                let csv = [];
                let headerRow = [];

                this.filteredRows.forEach((row, index) => {
                    let csvRow = [];

                    this.visibleColumns.forEach((column) => {
                        if (column.type !== 'button') {
                            if (index === 0) {
                                headerRow.push(column.name);
                            }

                            csvRow.push(this.renderCell(row, column.data));
                        }
                    });

                    csv.push(csvRow);
                });

                csv.unshift(headerRow);

                let file = 'data:text/csv;charset=utf-8,';
                csv.forEach((row) => {
                    file += encodeURIComponent(row.join(',') + '\r\n');
                });

                const link = document.createElement('a');
                link.setAttribute('href', file);
                link.setAttribute('download', 'csv-export.csv');
                document.body.appendChild(link);
                link.click();
                link.remove();
            }
        }
    }
</script>

<style lang="scss">
.coins-data-table-container {
    color: black;

    .coins-data-table-controls {

        button {
            margin-top: 10px;
            margin-bottom: 10px;
        }

        .coins-data-table-column-filter {
            background: lightgrey;
            border: 2px #0D6995 solid;
            border-radius: 5px;

            padding: 3px;

            display: inline-block;

            label {
                padding: 3px;
                padding-left: 5px;
            }
        }

        .coins-data-table-filter {
            padding-top: 10px;
            padding-bottom: 10px;
            float: right;
        }
    }

    table {
        box-shadow: 2px 2px 1px #888888;
        width: 100%;

        tr:nth-child(even) {
            background: lightgrey;
        }

        tr:nth-child(odd) {
            background: white;
        }

        tr {
            th {
                background: linear-gradient(#0D6995, #515151);
                color: white;
                border: 1px grey solid;

                padding: 7px 3px 7px 3px;

                cursor: pointer;

                white-space: nowrap;

                font-size: smaller;

                .coins-data-table-sort-icon {
                    margin: 5px;
                    border: 5px transparent solid;
                }

                .asc {
                    border-bottom: 5px red solid;
                }

                .desc {
                    border-top: 5px red solid;
                }
            }

            td {
                border: 1px grey solid;
                padding: 3px;

                div.pad {
                    padding: 5px 5px 5px 5px;
                }

                .tooltip-text {
                    opacity: 0;
                    transition: opacity 1s;

                    background-color: black;
                    color: #fff;

                    padding: 3px 3px 3px 3px;
                    margin-left: 2px;

                    border-radius: 5px;

                    position: absolute;
                    z-index: 1;
                }

                &:hover .tooltip-text {
                    opacity: 1;
                }

            }

            .a-checkbox label {
                padding-left: 10px;
                padding-right: 10px;
            }
        }
    }

    .coins-data-table-page-select {
        margin-top: 10px;
        text-align: center;

        .coins-data-table-pagination-controls {
            display: inline-block;
            min-width: 100px;
        }
    }

    button {
        margin: auto 5px auto 5px;

        background: lightgrey;

        display: inline-block;
        min-width: 35px;

        border-radius: 5px;
    }

    button:focus {
        outline:0;
    }

    button:hover {
        border: 2px #0D6995 solid;
    }

    button.current {
        border-bottom: 4px #0D6995 solid;
    }

    button.disabled {
        border: 2px white solid;
        opacity: .5;
        cursor: default;
    }

    .fade-enter-active, .fade-leave-active {
        transition: opacity .3s
    }
    .fade-enter, .fade-leave-to {
        opacity: 0
    }

    .left {
        float: left;
    }

    .right {
        float: right;
    }

    .spinner {
        border: 10px solid #515151; /* Blue */
        border-top: 10px solid #0D6995; /* Light grey */
        border-radius: 50%;

        width: 75px;
        height: 75px;

        margin: auto;
        margin-top: 30px;

        animation: spin 3s linear infinite;
    }

    @keyframes spin {
        0% { transform: rotate(0deg); }
        100% { transform: rotate(360deg); }
    }
}
</style>
