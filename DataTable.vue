<template>
  <div>
    <!-- Filter and controls -->
    <div class="row q-pb-md">
      <div class="col-12 col-md-8">
        <q-btn v-if="columnFilters.length > 0" flat round color="primary" size="sm" icon="fas fa-filter"
          @click="showFilterPanel = !showFilterPanel">
          <q-tooltip>Filtros da tabela</q-tooltip>
        </q-btn>
        <q-btn flat round color="primary" size="sm" icon="fas fa-columns">
          <q-tooltip>Colunas visíveis</q-tooltip>
          <q-menu class="q-pa-sm">
            <q-option-group v-model="visibleColumns" type="checkbox" :options="columnOptions"></q-option-group>
          </q-menu>
        </q-btn>
        <q-btn v-if="!!ExportXLS" flat round color="primary" size="sm" icon="fas fa-file-excel">
          <q-tooltip>Exportar para XLS</q-tooltip>
        </q-btn>
        <q-btn v-if="!!Printable" flat round color="primary" size="sm" icon="fas fa-print">
          <q-tooltip>Imprimir</q-tooltip>
        </q-btn>
      </div>
      <div class="col-12 col-md-4">
        <q-input dense square filled clearable label="Pesquisar na lista" v-model="searchTerm">
          <template v-slot:append>
            <q-icon size="xs" name="fas fa-search" color="grey-8" />
          </template></q-input>
      </div>
    </div>
    <q-separator></q-separator>
    <div v-if="columnFilters.length > 0" class="row">
      <div class="col-12">
        <q-expansion-item hide-expand-icon v-model="showFilterPanel" header-style="display:none;">
          <q-toolbar class="bg-grey-3">
            <q-toolbar-title>
              Filtros da Tabela
            </q-toolbar-title>
            <q-btn size="sm" icon="fas fa-filter-circle-xmark" color="primary" flat round dense
              @click="filterParams = {}">
              <q-tooltip>Limpar filtros</q-tooltip>
            </q-btn>
            <q-btn size="sm" icon="fas fa-close" color="primary" flat round dense @click="showFilterPanel = false">
              <q-tooltip>Fechar painel de filtros</q-tooltip>
            </q-btn>
          </q-toolbar>
          <div class="row q-py-sm">
            <div v-for="(f, i) in columnFilters" :key="i" class="col-12 col-md-4">
              <InputField clearable dense :type="f.type" :Label="`Filtrar por ${f.label}`" :Options="f.options ?? []"
                v-model="filterParams[f.field]">
              </InputField>
            </div>
          </div>
        </q-expansion-item>
      </div>
    </div>

    <q-separator></q-separator>

    <!-- Table -->
    <div class="datatable-container">
      <table>
        <thead>
          <tr>
            <th v-show="visibleColumns.includes(column.field) || column.name == 'actions'"
              :class="`q-pa-sm ${!!column.sort ? 'cursor-pointer' : ''}`" v-for="column in columns" :key="column.field"
              @click="sort(column)">
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
            <td v-show="visibleColumns.includes(column.field) || column.name == 'actions'"
              :class="`q-pa-sm ${(!!column.align) ? `text-${column.align}` : ''}`" v-for="column in columns"
              :key="column.field">

              <div v-if="column.name != 'actions'">
                <!-- In case no template is set for the td-->
                <div v-if="!(`cell-${column.name}` in $slots)">
                  {{ column.format ? column.format(row) : row[column.field] }}
                </div>

                <!-- In case a template is set for the td-->
                <div v-if="`cell-${column.name}` in $slots">
                  <slot :name="`cell-${column.name}`" :data="row"></slot>
                </div>
              </div>

              <!-- Especial td of actions -->
              <div class="text-center" v-if="column.name == 'actions' && showActions">
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
                  <q-icon size="lg" name="far fa-folder-open"></q-icon> *
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
    Name: {
      type: String,
      required: true
    },
    PaginationDefaults: Object,
    Columns: Object,
    RawData: Object,
    Actions: Object,
    LoadDataFn: Function,
    Filter: Function,
    ExportXLS: String,
    Printable: Boolean,
  },

  data() {
    return {
      pagination: {
        pages: [],
        currentPage: 1,
        finalPage: null,
        totalPages: null,
        totalItems: null,
        pageFirstItem: null,
        pageLastItem: null,
        pageLastIndex: null,
        limit: 10,
        sortBy: null,
        sortDir: 'ASC',
      },
      searchTerm: null,
      columns: [],
      visibleColumns: [],
      showFilterPanel: false,
      filterParams: {},
      filterTimeoutIndex: null,
      rowsInPage: [],
      loading: false,
      showLoader: true,
      loadTimeout: null,
      loadTimeout: null
    }
  },

  watch: {
    loading(isLoading) {
      this.showLoader = isLoading;
    },

    RawData: {
      handler(data) {
        this.paginate(data);
        // turn off loading indicator
        this.loading = false
      },
      deep: true
    },

    searchTerm() {
      this.showLoader = true;
      this.pagination.currentPage = 1;
      clearTimeout(this.loadTimeout);

      this.loadTimeout = setTimeout(() => {
        if (!!this.searchTerm)
          localStorage.setItem(`Datatable.${this.Name}.searchTerm`, this.searchTerm)
        else localStorage.removeItem(`Datatable.${this.Name}.searchTerm`)
        this.loadData()
      }, 200);
    },

    'pagination.currentPage'() {
      var persistedPagination = localStorage.getItem(`Datatable.${this.Name}.pagination`);
      persistedPagination = !!persistedPagination ? JSON.parse(persistedPagination) : {};
      persistedPagination.currentPage = this.pagination.currentPage;
      localStorage.removeItem(`Datatable.${this.Name}.pagination`)
      localStorage.setItem(`Datatable.${this.Name}.pagination`, JSON.stringify(persistedPagination))
      clearTimeout(this.loadTimeout);

      this.loadTimeout = setTimeout(this.loadData, 200);
    },

    'pagination.limit'() {
      this.pagination.currentPage = 1;
      var persistedPagination = localStorage.getItem(`Datatable.${this.Name}.pagination`);
      persistedPagination = !!persistedPagination ? JSON.parse(persistedPagination) : {};
      persistedPagination.currentPage = this.pagination.currentPage;
      persistedPagination.limit = this.pagination.limit;
      localStorage.removeItem(`Datatable.${this.Name}.pagination`)
      localStorage.setItem(`Datatable.${this.Name}.pagination`, JSON.stringify(persistedPagination))
      clearTimeout(this.loadTimeout);

      this.loadTimeout = setTimeout(this.loadData, 200);
    },

    'pagination.sortBy'() {
      var persistedPagination = localStorage.getItem(`Datatable.${this.Name}.pagination`);
      persistedPagination = !!persistedPagination ? JSON.parse(persistedPagination) : {};
      persistedPagination.sortBy = this.pagination.sortBy;
      localStorage.removeItem(`Datatable.${this.Name}.pagination`)
      localStorage.setItem(`Datatable.${this.Name}.pagination`, JSON.stringify(persistedPagination))
      clearTimeout(this.loadTimeout);

      this.loadTimeout = setTimeout(this.loadData, 200);
    },

    'pagination.sortDir'() {
      var persistedPagination = localStorage.getItem(`Datatable.${this.Name}.pagination`);
      persistedPagination = !!persistedPagination ? JSON.parse(persistedPagination) : {};
      persistedPagination.sortDir = this.pagination.sortDir;
      localStorage.removeItem(`Datatable.${this.Name}.pagination`)
      localStorage.setItem(`Datatable.${this.Name}.pagination`, JSON.stringify(persistedPagination))
      clearTimeout(this.loadTimeout);

      this.loadTimeout = setTimeout(this.loadData, 200);
    },

    visibleColumns(newVal) {
      if (newVal.length < 1) {
        this.visibleColumns = [this.Columns[0].field];
      }

      localStorage.setItem(`Datatable.${this.Name}.visibleColumns`, JSON.stringify(newVal));
    },

    filterParams: {
      handler() {
        // Save filters state:
        localStorage.removeItem(`Datatable.${this.Name}.filters`);

        if (Object.keys(this.filterParams).length > 0)
          localStorage.setItem(`Datatable.${this.Name}.filters`, JSON.stringify(this.filterParams));

        this.showLoader = true;
        this.pagination.currentPage = 1;
        clearTimeout(this.filterTimeoutIndex);

        for (let k in this.filterParams) {
          if (this.filterParams[k] == null)
            delete this.filterParams[k] == null
        }

        this.filterTimeoutIndex = setTimeout(() => {
          this.loadData()
        }, 200);
      },
      deep: true
    }

  },

  computed: {
    showActions() {
      for (let i = 0; i < this.Actions.length; i++) {
        let a = this.Actions[i];
        if (!!a.hide) {
          if (typeof a.hide == 'function') {
            for (let j = 0; j < this.rowsInPage.length; i++) {
              let row = this.rowsInPage[j];
              if (!a.hide(row)) return true;
            }
          } else return !a.hide;
        } else return true;
      }

      return false;
    },

    columnOptions() {
      return this.columns.map(clm => {
        return clm.name != 'actions' ? {
          label: clm.label,
          value: clm.field,
        } : null;
      }).filter(item => item != null);
    },

    columnFilters() {
      return this.Columns.map(clm => {
        if (!!clm.filter == false) return null;

        if (typeof clm.filter == 'string') {
          return {
            label: clm.label,
            field: clm.field,
            type: clm.filter
          };
        }

        return { label: clm.label, ...clm.filter };
      }).filter(item => item != null);
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

      // Search:
      if (this.searchTerm) {
        let f = null;
        var filterableColumns = [];
        for (let i = 0; i < this.columns.length; i++) {
          let column = this.columns[i];
          if (!column.field || column.field == '' || column.searchable === false) continue;
          if (column.field in this.filterParams) continue;

          filterableColumns.push(column);
        }

        for (let i = 0; i < filterableColumns.length; i++) {
          let column = filterableColumns[i];

          f = column.field;

          // First field:
          if (i == 0) {
            if (i == filterableColumns.length - 1) {
              result[f] = '$startFilterGroup$lkof$endFilterGroup|' + this.searchTerm;
            } else {
              result[f] = '$startFilterGroup$lkof|' + this.searchTerm;
            }
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

      var filterParams = {};
      // Handle filters:
      for (let k in this.filterParams) {
        let _f = this.columnFilters.find(x => x.field == k);
        if (_f.type == 'text')
          filterParams[k] = `$lkof|${this.filterParams[k]}`;
        else if (_f.type == 'daterange' || _f.type == 'datetimerange')
          filterParams[k] = `$btwn|${this.filterParams[k].from}|${this.filterParams[k].to}`;
        else filterParams[k] = this.filterParams[k]
      }

      result = {
        ...result, ...filterParams
      };
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

    paginate(data) {
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
    }
  },

  async mounted() {
    // Set columns:
    this.columns = [...this.Columns];
    this.visibleColumns = JSON.parse(localStorage.getItem(`Datatable.${this.Name}.visibleColumns`)) ?? this.Columns.map(clm => clm.field)
    if (this.Actions && this.Actions?.length > 0)
      this.columns.push({
        name: 'actions',
        label: 'Ações',
        align: 'center',
        sortable: false,
        filterable: false
      });

    // Set persisted filters:
    var persistedFilters = localStorage.getItem(`Datatable.${this.Name}.filters`);
    if (!!persistedFilters) {
      this.showFilterPanel = true;
      setTimeout(() => this.filterParams = JSON.parse(persistedFilters), 100)
    }

    // Set persisted search term:
    this.searchTerm = localStorage.getItem(`Datatable.${this.Name}.searchTerm`) ?? null;

    // Set sorting:
    for (let i = this.Columns.length - 1; i >= 0; i--) {
      let clm = this.Columns[i];
      if (!!clm.sortByDefault) {
        this.pagination.sortBy = clm.sort;
        this.pagination.sortDir = clm.sortByDefault;
        break;
      }

      if (!!clm.sort) {
        this.pagination.sortBy = clm.sort;
      }
    }

    // Set persisted pagination:
    var persistedPagination = localStorage.getItem(`Datatable.${this.Name}.pagination`);
    if (!!persistedPagination) {
      persistedPagination = JSON.parse(persistedPagination);
      for (let k in persistedPagination)
        if (k in this.pagination && k != 'currentPage')
          this.pagination[k] = persistedPagination[k]

      if (!!persistedPagination.currentPage && persistedPagination.currentPage != 1) {
        setTimeout(() => this.pagination.currentPage = persistedPagination.currentPage, 200);
      }
    }

    this.$emit('reload-fn', this.loadData);
  },
}
</script>

<style scoped>
.datatable-container {
  position: relative;
  overflow-x: scroll;
  width: 100%;
  max-height: 500px;
}

table {
  width: 100%;
  border-collapse: collapse;
}

thead {
  top: 0px;
  position: sticky;
  z-index: 2;
  background-color: white;
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
