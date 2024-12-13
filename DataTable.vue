<template>
  <div>
    <!-- Options and Controls -->
    <div class="row q-pb-md">
      <div class="col-12 col-md-8">
        <q-btn v-if="columnFilters.length > 0" flat round color="primary" size="sm" icon="fas fa-filter"
          @click="showFilterPanel = !showFilterPanel">
          <q-tooltip>Filtros da tabela</q-tooltip>
        </q-btn>
        <q-btn flat round color="primary" size="sm" icon="fas fa-sync" @click="reload()">
          <q-tooltip>Atualizar lista</q-tooltip>
        </q-btn>
        <q-btn flat round color="primary" size="sm" icon="fas fa-columns">
          <q-tooltip>Colunas visíveis</q-tooltip>
          <q-menu class="q-pa-sm">
            <q-option-group v-model="visibleColumns" type="checkbox" :options="columnOptions"></q-option-group>
          </q-menu>
        </q-btn>
        <q-btn v-if="!!Export" flat round color="primary" size="sm" icon="fas fa-file-download">
          <q-tooltip>Opções de Exportação</q-tooltip>
          <q-menu class="q-pa-sm">
            <q-list class="text-primary">
              <q-item v-close-popup dense v-for="(opt, idx) in exportOptions" :key="idx" clickable
                @click="exportFile(opt.filetype, opt.filename)">
                <q-item-section avatar>
                  <q-icon :name="opt.icon" size="xs"></q-icon>
                </q-item-section>
                <q-item-section>
                  {{ opt.label }}
                </q-item-section>
              </q-item>
            </q-list>
          </q-menu>
        </q-btn>
        <q-btn v-if="!!Printable" flat round color="primary" size="sm" icon="fas fa-print" @click="printData()">
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

    <!-- Filters Panel -->
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

        <!-- Ready State -->
        <tbody v-if="state == 'ready'">
          <tr v-for="(row, idx) in dataInPage" :key="idx">
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
                      <q-item v-show="typeof a.hide == 'function' ? !a.hide(row) : !a.hide"
                        v-for="(a, idx) in RowActions" :key="idx" clickable v-close-popup @click="a.fn(row)">
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

        <!-- Empty State -->
        <tbody v-if="state == 'empty'">
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

        <!-- Loading State -->
        <tbody v-if="state == 'loading'">
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
    <div class="row q-mt-lg" v-show="state == 'ready'">
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
// Services:
import Http from '../../../services/http'
import Utils from '../../../services/utils'

export default {
  name: 'ui-toolcase-gadgets-datatable',

  props: {
    Name: {
      type: String,
      required: true
    },
    modelValue: {
      type: Object,
      required: true
    },
    DataURL: {
      type: String,
      required: true
    },
    Columns: {
      type: Object,
      required: true
    },
    RowActions: Object,
    ExtraFilters: Object,
    Export: Object,
    Printable: Boolean,
    BeforeLoad: Function,
    OnLoaded: Function,
  },

  data() {
    return {
      // Pagination related vars:
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

      // Filters related vars:
      searchTerm: null,
      showFilterPanel: false,
      filterParams: {},

      // Columns settings:
      visibleColumns: [],
      columns: [],

      // Data:
      rawData: [],
      dataInPage: [],
      loadTimeout: null,

      // State:
      loading: false,
      showLoader: true,
      errorState: false
    }
  },

  watch: {
    searchTerm() {
      this.showLoader = true;
      this.pagination.currentPage = 1;
      clearTimeout(this.loadTimeout);

      this.loadTimeout = setTimeout(async () => {
        if (!!this.searchTerm)
          localStorage.setItem(`Datatable.${this.Name}.searchTerm`, this.searchTerm)
        else localStorage.removeItem(`Datatable.${this.Name}.searchTerm`)

        this.rawData = (await this.loadData()).data;
      }, 200);
    },

    'pagination.currentPage'() {
      var persistedPagination = localStorage.getItem(`Datatable.${this.Name}.pagination`);
      persistedPagination = !!persistedPagination ? JSON.parse(persistedPagination) : {};
      persistedPagination.currentPage = this.pagination.currentPage;
      localStorage.removeItem(`Datatable.${this.Name}.pagination`)
      localStorage.setItem(`Datatable.${this.Name}.pagination`, JSON.stringify(persistedPagination))
      clearTimeout(this.loadTimeout);

      this.loadTimeout = setTimeout(async () => this.rawData = (await this.loadData()).data, 200);
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

      this.loadTimeout = setTimeout(async () => this.rawData = (await this.loadData()).data, 200);
    },

    'pagination.sortBy'() {
      var persistedPagination = localStorage.getItem(`Datatable.${this.Name}.pagination`);
      persistedPagination = !!persistedPagination ? JSON.parse(persistedPagination) : {};
      persistedPagination.sortBy = this.pagination.sortBy;
      localStorage.removeItem(`Datatable.${this.Name}.pagination`)
      localStorage.setItem(`Datatable.${this.Name}.pagination`, JSON.stringify(persistedPagination))
      clearTimeout(this.loadTimeout);

      this.loadTimeout = setTimeout(async () => this.rawData = (await this.loadData()).data, 200);
    },

    'pagination.sortDir'() {
      var persistedPagination = localStorage.getItem(`Datatable.${this.Name}.pagination`);
      persistedPagination = !!persistedPagination ? JSON.parse(persistedPagination) : {};
      persistedPagination.sortDir = this.pagination.sortDir;
      localStorage.removeItem(`Datatable.${this.Name}.pagination`)
      localStorage.setItem(`Datatable.${this.Name}.pagination`, JSON.stringify(persistedPagination))
      clearTimeout(this.loadTimeout);

      this.loadTimeout = setTimeout(async () => this.rawData = (await this.loadData()).data, 200);
    },

    filterParams: {
      handler(v) {
        this.filterHandler(v, 'filters')
      },
      deep: true
    },

    ExtraFilters: {
      handler(v) {
        this.filterHandler(v, 'extrafilters');
      },
      deep: true
    },

    visibleColumns(newVal) {
      if (newVal.length < 1) {
        this.visibleColumns = [this.Columns[0].field];
      }

      localStorage.setItem(`Datatable.${this.Name}.visibleColumns`, JSON.stringify(newVal));
    },

    loading(isLoading) {
      this.showLoader = isLoading;
    },

    rawData: {
      handler(data) {
        this.paginate(data);
        // Expose factory:
        this.exposeFactory();
        // turn off loading indicator
        this.loading = false
      },
      deep: true
    },
  },

  computed: {
    showActions() {
      for (let i = 0; i < this.RowActions.length; i++) {
        let a = this.RowActions[i];
        if (!!a.hide) {
          if (typeof a.hide == 'function') {
            for (let j = 0; j < this.dataInPage.length; i++) {
              let row = this.dataInPage[j];
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
    },

    exportOptions() {
      var typeIcons = {
        xls: 'fas fa-file-excel',
      };
      var typeLabels = {
        xls: 'Exportar XLS',
      };

      if (!(this.Export instanceof Array))
        return [{
          ...this.Export,
          icon: typeIcons[this.Export.filetype],
          label: typeLabels[this.Export.filetype]
        }];

      return this.Export.map((opt) => ({
        ...opt,
        icon: typeIcons[opt.filetype],
        label: typeLabels[opt.filetype]
      }));
    },

    state() {
      if (this.showLoader) return 'loading';
      if (this.errorState) return 'error';
      if (this.dataInPage.length > 0) return 'ready';
      if (this.dataInPage.length == 0) return 'empty';
      return null;
    },
  },

  methods: {
    filterHandler(filtersObject, name) {
      // Save filters state:
      localStorage.removeItem(`Datatable.${this.Name}.${name}`);

      if (Object.keys(filtersObject).length > 0)
        localStorage.setItem(`Datatable.${this.Name}.${name}`, JSON.stringify(filtersObject));

      this.showLoader = true;
      this.pagination.currentPage = 1;
      clearTimeout(this.loadTimeout);

      for (let k in filtersObject) {
        if (filtersObject[k] == null)
          delete filtersObject[k] == null
      }

      this.loadTimeout = setTimeout(async () => this.rawData = (await this.loadData()).data, 200);
    },

    setParams() {
      // Pagination Params:
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

      result = { ...result, ...filterParams };

      var extraFilters = {};
      if (!!this.ExtraFilters)
        for (let k in this.ExtraFilters)
          if (!!this.ExtraFilters[k])
            extraFilters[k] = this.ExtraFilters[k];

      result = { ...result, ...extraFilters };

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

    reload() {
      clearTimeout(this.loadTimeout);

      this.loadTimeout = setTimeout(async () => {
        this.rawData = (await this.loadData()).data;
      }, 200);
    },

    async loadData(ignorePagination) {
      if (!this.loading) {
        // turn on loading indicator
        this.loading = true;

        var params = this.setParams();
        if (!!ignorePagination) {
          delete params.$page;
          delete params.$limit;
        }

        try {
          // Before Load callback:
          if (this.BeforeLoad) await this.BeforeLoad(params);

          // fetch data from server
          var response = await Http.get(this.DataURL, params);

          // On Loaded callback:
          if (this.OnLoaded) await this.OnLoaded(response);

          return response;
        } catch (err) {
          this.loading = false;
          this.errorState = true;
          this.$emit('error-thrown', err);
        }
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

      this.dataInPage = [];
      for (let i = 0; i <= this.pagination.pageLastIndex; i++) {
        this.dataInPage.push(this.rawData[i]);
      }

      if (this.dataInPage.length == 0 && this.pagination.currentPage > 1) {
        this.goToPage('prev');
      }
    },

    async exportFile(filetype, filename) {
      filename = filename.indexOf(`.${filetype}`) ? filename : `${filename}.${filetype}`;

      var data = (await this.loadData(true)).data;
      var blobType;
      var content;
      switch (filetype) {
        case 'xls':
          blobType = "application/vnd.ms-excel;charset=utf-8;";
          content = this.buildContentTable(data);
          break;
      }

      // Cria um blob com o conteúdo do arquivo
      const blob = new Blob([content], { type: blobType });

      const link = document.createElement("a");
      const url = URL.createObjectURL(blob);
      link.setAttribute("href", url);
      link.setAttribute("download", filename);

      link.style.visibility = "hidden";

      // Adiciona o link ao DOM e dispara o clique
      document.body.appendChild(link);
      link.click();
      document.body.removeChild(link);

      this.loading = false;
    },

    async printData() {
      var data = (await this.loadData(true)).data;
      this.loading = false;

      const content = `${this.buildContentTable(data)}`;

      // Open a new window or tab for printing
      const newWindow = window.open();
      newWindow.document.open();
      newWindow.document.write(content);
      newWindow.document.close();
      newWindow.print();
    },

    exposeFactory() {
      this.$emit('update:model-value', {
        state: this.state,
        params: this.setParams(),
        rawData: this.rawData,
        dataInPage: this.dataInPage,
        visibleColumns: this.visibleColumns,
        reload: this.reload
      });
    },

    escapeXml(value) {
      if (value === null || value === undefined) return "";
      return String(value)
        .replace(/&/g, "&amp;")
        .replace(/</g, "&lt;")
        .replace(/>/g, "&gt;")
        .replace(/"/g, "&quot;")
        .replace(/'/g, "&#39;");
    },

    buildContentTable(data) {
      // Início do XML
      let content = `
      <?xml version="1.0" encoding="UTF-8"?>
  <html xmlns:o="urn:schemas-microsoft-com:office:office"
        xmlns:x="urn:schemas-microsoft-com:office:excel"
        xmlns="http://www.w3.org/TR/REC-html40">
    <head>
      <style>
        table {
          border-collapse: collapse;
          width: 100%;
        }
        th, td {
          border: 1px solid #000;
          padding: 5px;
          text-align: left;
        }
        th {
          background-color: #f4f4f4;
          text-align: center;
        }
      </style>
      <meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
    </head>
    <body>
  <table>
    <thead>
      <tr>
        ${this.Columns
          .filter(column => this.visibleColumns.includes(column.field))
          .map(column => `<th>${this.escapeXml(column.label)}</th>`).join("")}
      </tr>
    </thead>
    <tbody>
  `;

      // Adiciona as linhas
      for (let j = 0; j < data.length; j++) {
        content += '<tr>';
        let row = data[j];

        for (let i = 0; i < this.Columns.length; i++) {
          let clm = this.Columns[i];
          if (!(this.visibleColumns.includes(clm.field))) continue;
          content += `<td>${row[clm.field]}</td>`;
        }
        content += '</tr>';
      }
      // Fecha o XML
      content += `
    </tbody>
  </table>
  `;

      return content;
    }
  },

  async mounted() {
    // Set columns:
    this.columns = [...this.Columns];
    this.visibleColumns = JSON.parse(localStorage.getItem(`Datatable.${this.Name}.visibleColumns`)) ?? this.Columns.map(clm => clm.field)
    if (this.RowActions && this.RowActions?.length > 0)
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
