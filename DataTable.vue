<template>
  <div>
    <!--Search Input-->
    <div class="row justify-end">
      <div class="col-12 col-md-4">
        <q-input square filled clearable label="Pesquisar na lista" v-model="searchTerm">
          <template v-slot:append>
            <q-icon size="xs" name="fas fa-search" color="grey-8" />
          </template></q-input>
      </div>
    </div>

    <!-- Table -->
    <div class="datatable-container">
      <table>
        <thead>
          <tr>
            <th :class="`q-pa-sm ${!!column.sort ? 'cursor-pointer' : ''}`" v-for="column in columns"
              :key="column.field" @click="sort(column)">
              {{ column.label }}
              <q-icon v-if="!!column.sort" size="0.9em" :name="getSortIcon(column)"
                :color="column.sort == this.pagination.sortBy ? 'primary' : null"></q-icon>
              <q-tooltip v-if="!!column.sort">Clique para ordenar p/ {{ column.label }}</q-tooltip>
            </th>
          </tr>
        </thead>

        <!-- Filled -->
        <tbody v-if="rowsInPage.length > 0 && !showLoader">
          <tr v-for="(row, idx) in rowsInPage" :key="idx">
            <td :class="`q-pa-sm ${(!!column.align) ? `text-${column.align}` : ''}`" v-for="column in columns"
              :key="column.field">

              <div v-if="column.name != 'acoes'">
                <!-- In case no template is set for the td-->
                <div v-if="!(`cell-${column.name}` in $slots)">
                  {{ row[column.field] }}
                </div>

                <!-- In case a template is set for the td-->
                <div v-if="`cell-${column.name}` in $slots">
                  <slot :name="`cell-${column.name}`" :data="row"></slot>
                </div>
              </div>

              <!-- Especial td of actions -->
              <div class="text-center" v-if="column.name == 'acoes'">
                <q-btn flat dense color="primary" icon="fas fa-ellipsis-v">
                  <q-tooltip>Ações do registro</q-tooltip>
                  <q-menu>
                    <q-list>
                      <q-item v-show="typeof a.hide == 'function' ? !a.hide(row) : !a.hide" v-for="(a, idx) in Actions"
                        :key="idx" clickable v-close-popup @click="a.fn(row)">
                        <q-item-section v-if="a.icon" side>
                          <q-icon size="sm" :name="a.icon"></q-icon>
                        </q-item-section>
                        <q-item-section>
                          {{ a.label }}
                        </q-item-section>
                        <q-tooltip v-if="a.tooltip">{{ a.tooltip }}</q-tooltip>
                      </q-item>
                    </q-list>
                  </q-menu>
                </q-btn>
              </div>
            </td>
          </tr>
        </tbody>

        <!-- Empty -->
        <tbody v-if="rowsInPage.length == 0 && !showLoader">
          <tr>
            <td class="q-pa-lg text-center" :colspan="columns.length">
              <div>
                <div>
                  <q-icon size="lg" name="fas fa-folder-open"></q-icon> *
                </div>
                <div class="text-h6">
                  Lista vazia.
                </div>
              </div>
            </td>
          </tr>
        </tbody>

        <!-- Loading -->
        <tbody v-if="showLoader">
          <tr>
            <td class="q-pa-lg text-center" :colspan="columns.length">
              <div>
                <q-spinner-oval size="lg" />
              </div>
              <div class="text-caption">
                Carregando...
              </div>
            </td>
          </tr>
        </tbody>

      </table>

    </div>

    <!-- Pagination -->
    <div class="row q-mt-lg" v-show="rowsInPage.length > 0 && !showLoader">
      <div :class="`col-12 col-md-6 ${$q.screen.lt.md ? 'text-center' : ''}`">
        <div>
          Mostrar
          <select class="q-pa-xs" v-model="pagination.limit">
            <option value="5">5</option>
            <option value="10">10</option>
            <option value="25">25</option>
            <option value="50">50</option>
          </select>
          registros
        </div>
      </div>
      <div :class="`col-12 col-md-6 ${$q.screen.gt.sm ? 'text-right' : 'text-center q-mt-md'}`">
        <q-btn color="primary" @click="goToPage('prev')">
          <q-tooltip>Página Anterior</q-tooltip>
          <q-icon size="xs" name="fas fa-chevron-left"></q-icon>
        </q-btn>
        <q-btn class="q-px-sm" color="primary" v-for="page in pagination.pages" :key="page" @click="goToPage(page)"
          :flat="page == pagination.currentPage">
          <q-tooltip>Página {{ page }}</q-tooltip>
          {{ page }}
        </q-btn>
        <q-btn color="primary" @click="goToPage('next')">
          <q-tooltip>Próxima Página</q-tooltip>
          <q-icon size="xs" name="fas fa-chevron-right"></q-icon>
        </q-btn>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'datatable-component',

  props: {
    Name: String,
    PaginationDefaults: Object,
    Settings: Object,
    Columns: Object,
    RawData: Object,
    Actions: Object,
    LoadDataFn: Function,
    Filter: Function
  },

  data() {
    return {
      pagination: {
        pages: [],
        currentPage: null,
        finalPage: null,
        totalPages: null,
        totalItems: null,
        pageFirstItem: null,
        pageLastItem: null,
        pageLastIndex: null,
        limit: null,
        sortBy: null,
        sortDir: null,
      },
      searchTerm: null,
      columns: [],
      rowsInPage: [],
      loading: false,
      showLoader: false,
      searchTimeout: null,
      loadTimeout: null
    }
  },

  methods: {
    setParams() {
      var result = {
        '$sort_by': this.pagination.sortBy,
        '$sort_direction': this.pagination.sortDir,
        '$page': this.pagination.currentPage,
        '$limit': Number(this.pagination.limit)
      };

      if (this.searchTerm) {
        let f = null;
        var filterableColumns = [];
        for (let i = 0; i < this.columns.length; i++) {
          let column = this.columns[i];
          if (!column.field || column.field == '' || column.filterable === false) continue;

          filterableColumns.push(column);
        }

        for (let i = 0; i < filterableColumns.length; i++) {
          let column = filterableColumns[i];

          f = column.field;

          // First field:
          if (i == 0) {
            result[f] = '$startFilterGroup$lkof|' + this.searchTerm;
          }
          // All fields in the middle:
          else if (i < (filterableColumns.length - 1)) {
            result[f] = '$or$lkof|' + this.searchTerm;
          }
          // Last field:
          else {
            result[f] = '$endFilterGroup$or$lkof|' + this.searchTerm;
          }
        }
      }

      if (!!this.Filter) result = { ...this.Filter(), ...result };

      this.$emit('params-changed', result);

      return result;
    },

    getSortIcon(column) {
      if (column.sort == this.pagination.sortBy) {
        if (this.pagination.sortDir == 'ASC') return 'fas fa-sort-up';
        else if (this.pagination.sortDir == 'DESC') return 'fas fa-sort-down';
        else return 'fas fa-ban';
      } else return 'fas fa-sort';
    },

    sort(column) {
      if (!!column.sort === false) return;

      if (this.pagination.sortBy == column.sort) {
        if (this.pagination.sortDir == 'ASC') this.pagination.sortDir = 'DESC';
        else if (this.pagination.sortDir == 'DESC') this.pagination.sortDir = 'ASC';

      } else {
        this.pagination.sortBy = column.sort;
        this.pagination.sortDir = 'ASC';
      }

    },

    async goToPage(page) {
      switch (page) {
        case 'next':
          if ((this.pagination.currentPage + 1) > this.pagination.finalPage) return;
          this.pagination.currentPage++;
          break;
        case 'prev':
          if ((this.pagination.currentPage - 1) < 1) return;
          this.pagination.currentPage--;
          break;
        default:
          if (page != this.pagination.currentPage)
            this.pagination.currentPage = page;
      }
    },

    async loadData() {
      if (!this.loading) {
        // turn on loading indicator
        this.loading = true;

        // fetch data from server
        await this.LoadDataFn(this.setParams())
      }
    },
  },

  watch: {
    loading(isLoading) {
      this.showLoader = isLoading;
    },

    RawData: {
      handler(data) {
        this.pagination.finalPage = this.pagination.currentPage + (Math.ceil(data.length / this.pagination.limit) - 1);

        var initial = null;

        if (this.pagination.finalPage - this.pagination.currentPage < 1) {
          initial = this.pagination.currentPage - 4;
        } else if (this.pagination.finalPage - this.pagination.currentPage < 2) {
          initial = this.pagination.currentPage - 3;
        } else {
          initial = this.pagination.currentPage - 2;
        }
        initial = initial < 1 ? 1 : initial;

        this.pagination.pages = [];
        for (let i = initial; i <= this.pagination.finalPage; i++) {
          this.pagination.pages.push(i);
          if (this.pagination.pages.length > 4) break;
        }

        this.pagination.pageFirstItem = data.length > 0 ? (((this.pagination.currentPage * this.pagination.limit) - this.pagination.limit) + 1) : 0;

        if (data.length > 0) {
          this.pagination.pageLastItem = this.pagination.pageFirstItem;
          for (let i = 1; i < data.length; i++) {
            this.pagination.pageLastItem++;
          }
        } else this.pagination.pageLastItem = 0;

        this.pagination.pageLastIndex = data.length < this.pagination.limit ? data.length - 1 : this.pagination.limit - 1;

        this.rowsInPage = [];
        for (let i = 0; i <= this.pagination.pageLastIndex; i++) {
          this.rowsInPage.push(this.RawData[i]);
        }

        if (this.rowsInPage.length == 0 && this.pagination.currentPage > 1) {
          this.goToPage('prev');
        }
        // turn off loading indicator
        this.loading = false
      },
      deep: true
    },

    searchTerm() {
      this.showLoader = true;
      this.pagination.currentPage = 1;
      clearTimeout(this.searchTimeout);

      this.searchTimeout = setTimeout(this.loadData, 300);
    },

    async 'pagination.currentPage'() {
      await this.loadData()
    },

    async 'pagination.limit'() {
      this.pagination.currentPage = 1;
      await this.loadData()
    },

    async 'pagination.sortBy'() {
      await this.loadData()
    },

    async 'pagination.sortDir'() {
      await this.loadData()
    },

  },

  created() {
    // Set columns:
    this.columns = [...this.Columns];
    if (this.Actions && this.Actions?.length > 0)
      this.columns.push({
        name: 'acoes',
        label: 'Ações',
        align: 'center',
        sortable: false,
        filterable: false
      });

    // Set default pagination:
    this.pagination.currentPage = this.PaginationDefaults?.currentPage ? this.PaginationDefaults?.currentPage : 1;
    this.pagination.limit = this.PaginationDefaults?.limit ? this.PaginationDefaults?.limit : 10;
    this.pagination.sortBy = this.PaginationDefaults?.sortBy ? this.PaginationDefaults?.sortBy : '1';
    this.pagination.sortDir = this.PaginationDefaults?.sortDir ? this.PaginationDefaults?.sortDir : 'ASC';

    this.$emit('reload-fn', this.loadData);
  },
}
</script>

<style scoped>
.datatable-container {
  overflow-x: scroll;
  width: 100%;
}

table {
  width: 100%;
  border-collapse: collapse;
}

tbody>tr:nth-child(even) {
  background-color: #e2e2e2;
}

select {
  cursor: pointer;
}

.cursor-pointer {
  cursor: pointer;
}
</style>
