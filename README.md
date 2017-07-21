# coins-vue-data-table
A Vue.js data table component

## Properties

##### `name` ( type ) [ default ]

##### `data-source` (string | array) [ required ]
If `data-source` is a string, a `GET` request will be made to that URL. Otherwise, if `data-source` is an array, that will be used as the table data.

##### `columns` (array) [ required ]
An array of the table columns an their parameters. Possible values:
- `name` Name displayed on table
- `data` (string) Key referenced in data
- `data` (function) A function determines column render. It is passed the data row.
- `type` Currently the only supported option is 'numeric'. This facilitates accurate sorting. By default sorting uses String's `localeCompare` function.
- `visible` Whether or not a column should be initially visible

Note: By default boolean table values are converted to Yes/No for `true`/`false` respectively. This behavior can be overwritten with the function `data` data type.

### Pagination

##### `rowsPerPage` (integer) [ 20 ]
Number of rows to display

##### `visiblePageRange` (integer) [ 6 ]
Number of pages to be visible in the page selector

##### `skipRange` (integer) [ 5 ]
How far the `>>` and `<<` buttons should skip

### Table Controls

##### `exportButton` (boolean) [ `true` ]
Whether or not the export button should be visible

##### `columnsButton` ( boolean ) [ `true` ]
Whether or not the column selector button should be visible

##### `filter` ( boolean ) [ `true` ]
Whether or not the filter should be visible
