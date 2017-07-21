<template>
  <div class="coins-data-table-container">
    <div v-show="!loading" class="coins-data-table-controls">
      <button v-if="exportButton">Export CSV</button>

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

      <label v-if="filter" class="coins-data-table-filter">
        Filter:&nbsp
        <input v-model="filterText"></input>
      </label>
    </div>

    <table id="coins-data-table">
      <thead>
        <tr>
          <th v-for="column in visibleColumns" @click="updateSort(column)">
            {{ column.name }}
            <div class="coins-data-table-sort-icon right"
                 :class="{ asc: column.sort === 'asc', desc: column.sort === 'desc' }">
            </div>
          </th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="row in displayedRows">
          <td v-for="column in visibleColumns">{{ renderCell(row, column.data) }}</td>
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
                visibleColumnNames: []
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
                            if (this.renderCell(row, column.data)) {
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
                    return filtered.sort((a, b) => {
                        a = this.renderCell(a, this.sort.name) || '';
                        b = this.renderCell(b, this.sort.name) || '';

                        if (this.sort.type === 'numeric') {
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
            }
        },

        mounted() {
            if (this.dataSource instanceof Array) {
                this.data = this.dataSource;
                this.loading = false;
                return;
            }

            // URL (string) given
            axios.get(this.dataSource)
                .then(({ data: { data } }) => { this.data = data; this.loading = false; })
                .catch(() => { alert('There was an error getting the table data.'); });

            this.columns.forEach((column) => {
                this.visibleColumnNames.push(column.name);
            });
        },

        methods: {
            renderCell(row, data) {
                // Prioritize function types
                if (typeof data === 'function') {
                    return data(row)
                }

                // Booleans default to true -> Yes / false-> No
                if (typeof row[data] === 'boolean') {
                    return row[data] ? 'Yes' : 'No';
                }

                return row[data];
            },
            updateSort(column) {
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