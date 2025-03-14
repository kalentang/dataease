<script>
import { filter, forEach, find, split, get } from 'lodash-es'
import { listDatasource, listDatasourceType } from '@/api/system/datasource'
import { listForm, saveForm } from '@/views/dataFilling/form/dataFilling'

export default {
  name: 'DataFillingFormSave',
  props: {
    form: {
      type: Object,
      required: true
    },
    showDrawer: {
      type: Boolean,
      required: true
    }
  },
  data: function() {
    const checkDuplicateNameValidator = (rule, value, callback) => {
      if (!value) {
        return callback(new Error(this.$t('commons.component.required')))
      }
      let count = 0
      forEach(this.formData.forms, f => {
        if (f.type === 'dateRange') {
          if (f.settings.mapping.columnName1 === value) {
            count++
          }
          if (f.settings.mapping.columnName2 === value) {
            count++
          }
        } else {
          if (f.settings.mapping.columnName === value) {
            count++
          }
        }
      })
      if (count > 1) {
        callback(new Error(this.$t('data_fill.form.duplicate_error')))
      }
      callback()
    }
    const checkDuplicateIndexNameValidator = (rule, value, callback) => {
      if (!value) {
        return callback(new Error(this.$t('commons.component.required')))
      }
      let count = 0
      forEach(this.formData.tableIndexes, f => {
        if (f.name === value) {
          count++
        }
      })
      if (count > 1) {
        callback(new Error(this.$t('data_fill.form.duplicate_error')))
      }
      callback()
    }
    const checkInvalidColumnValidator = (rule, value, callback) => {
      if (!value) {
        return callback(new Error(this.$t('commons.component.required')))
      }
      if (this.columnsList.length === 0) {
        return callback(new Error(this.$t('data_fill.form.value_not_exists')))
      }
      if (find(this.columnsList, c => c === value) === undefined) {
        callback(new Error(this.$t('data_fill.form.value_not_exists')))
      }
      callback()
    }
    const checkDuplicateIndexColumnValidator = (rule, value, callback, source) => {
      if (!value) {
        return callback(new Error(this.$t('commons.component.required')))
      }
      const f = split(rule.field, '.')[0]
      const _list = get(this.formData, f)

      let count = 0
      forEach(_list.columns, f => {
        if (f.column === value) {
          count++
        }
      })
      if (count > 1) {
        callback(new Error(this.$t('data_fill.form.duplicate_error')))
      }
      callback()
    }
    return {
      loading: false,
      requiredRule: { required: true, message: this.$t('commons.required'), trigger: ['blur', 'change'] },
      duplicateRule: { validator: checkDuplicateNameValidator, trigger: 'blur' },
      duplicateIndexRule: { validator: checkDuplicateIndexNameValidator, trigger: 'blur' },
      duplicateIndexColumnRule: { validator: checkDuplicateIndexColumnValidator, trigger: ['blur', 'change'] },
      invalidColumnRule: { validator: checkInvalidColumnValidator, trigger: ['blur', 'change'] },
      formData: {},
      folders: [],
      allDatasourceTypes: [],
      allDatasourceList: [],
      folderTreeShow: false
    }
  },
  computed: {
    datasourceList() {
      const _types = filter(this.allDatasourceTypes, t => t.type === 'mysql' || t.type === 'mariadb')
      forEach(_types, t => {
        t.options = filter(this.allDatasourceList, d => d.type === t.type)
      })
      return _types
    },
    selectDatasets() {
      const result = []
      this.flattenFolder(this.folders, result)
      return result
    },
    columnsList() {
      const _list = []
      for (let i = 0; i < this.formData.forms.length; i++) {
        const row = this.formData.forms[i]
        if (row.type === 'dateRange') {
          if (row.settings.mapping.columnName1 !== undefined && row.settings.mapping.columnName1 !== '') {
            _list.push(row.settings.mapping.columnName1)
          }
          if (row.settings.mapping.columnName2 !== undefined && row.settings.mapping.columnName2 !== '') {
            _list.push(row.settings.mapping.columnName2)
          }
        } else {
          if (row.settings.mapping.columnName !== undefined && row.settings.mapping.columnName !== '' && row.settings.mapping.type !== 'text') {
            _list.push(row.settings.mapping.columnName)
          }
        }
      }

      return _list
    }
  },
  watch: {
    formData: {
      handler(newVal, oldVal) {
        this.$emit('update:form', newVal)
      },
      deep: true
    }
  },
  mounted() {
    this.loading = true
    this.formData = this.form

    forEach(this.formData.forms, f => {
      f.settings.mapping.typeOptions = this.getTypeOptions(f)
      if (!f.settings.mapping.type) {
        f.settings.mapping.type = f.settings.mapping.typeOptions[0].value
      }
    })
    const p1 = listDatasourceType()
    const p2 = listDatasource()
    const p3 = listForm({ nodeType: 'folder' })

    Promise.all([p1, p2, p3]).then((val) => {
      this.allDatasourceTypes = val[0].data

      this.allDatasourceList = val[1].data

      this.folders = val[2].data || []
      if (this.formData.folder) {
        this.$nextTick(() => {
          this.$refs.tree.setCurrentKey(this.formData.folder)
          this.$refs.tree.setCheckedKeys([this.formData.folder])
        })
      }
    }).finally(() => {
      this.loading = false
    })
  },
  methods: {
    getTypeOptions(formOption) {
      const _options = []
      if (formOption.type !== 'date' &&
        formOption.type !== 'dateRange' &&
        formOption.settings.inputType !== 'number' &&
        formOption.type !== 'textarea' &&
        formOption.type !== 'checkbox' &&
        !(formOption.type === 'select' && formOption.settings.multiple)
      ) {
        _options.push({ value: 'nvarchar', label: this.$t('data_fill.database.nvarchar') })
      }
      if (formOption.type === 'checkbox' ||
        formOption.type === 'select' && formOption.settings.multiple ||
        formOption.type === 'textarea') {
        _options.push({ value: 'text', label: this.$t('data_fill.database.text') })
      }

      if (formOption.type === 'input' && formOption.settings.inputType === 'number') {
        _options.push({ value: 'number', label: this.$t('data_fill.database.number') })
        _options.push({ value: 'decimal', label: this.$t('data_fill.database.decimal') })
      }
      if (formOption.type === 'date' || formOption.type === 'dateRange') {
        _options.push({ value: 'datetime', label: this.$t('data_fill.database.datetime') })
      }
      return _options
    },
    flattenFolder(list, result = []) {
      forEach(list, item => {
        result.push(item)
        if (item.children && item.children.length > 0) {
          this.flattenFolder(item.children, result)
        }
      })
      return result
    },
    closeSave() {
      this.$emit('update:showDrawer', false)
    },
    validateForm() {
      this.$nextTick(() => {
        this.$refs['mRightForm'].validate()
      })
    },
    addIndex() {
      this.formData.tableIndexes.push({
        name: undefined,
        columns: [{
          column: undefined,
          order: 'none'
        }]
      })
    },
    removeIndex(index) {
      this.formData.tableIndexes.splice(index, 1)
    },
    removeIndexColumn(list, index) {
      list.splice(index, 1)
    },
    addColumn(list) {
      list.push({
        column: undefined,
        order: 'none'
      })
    },
    nodeClick(data) {
      this.$nextTick(() => {
        this.formData.folder = data.id
        this.formData.level = data.level + 1
        this.folderTreeShow = false
      })
    },
    filterMethod(val) {
      if (!val) this.$refs.tree.filter(val)
      this.$refs.tree.filter(val)
    },
    filterNode(value, data) {
      if (!value) return true
      return data.name.indexOf(value) !== -1
    },
    doSave() {
      this.loading = true
      this.$refs['mRightForm'].validate((valid) => {
        if (valid) {
          const data = {
            name: this.formData.name,
            tableName: this.formData.table,
            datasource: this.formData.datasource,
            pid: this.formData.folder,
            level: this.formData.level,
            forms: JSON.stringify(this.formData.forms),
            createIndex: this.formData.createIndex,
            tableIndexes: JSON.stringify(this.formData.tableIndexes),
            nodeType: 'form'
          }
          saveForm(data).then(res => {
            this.closeSave()
            this.$router.replace({ name: 'data-filling-form', query: { id: res.data }})
          }).finally(() => {
            this.loading = false
          })
        } else {
          this.loading = false
          return false
        }
      })
    }
  }
}
</script>

<template>
  <el-container
    v-loading="loading"
    class="DataFillingFormSave"
  >
    <el-header class="de-header">
      <div class="panel-info-area">
        <span class="text16 margin-left12">
          {{ $t('data_fill.form.save_form') }}
        </span>
      </div>

      <div style="padding-right: 20px">
        <i
          class="el-icon-close"
          style="cursor: pointer"
          @click="closeSave"
        />
      </div>
    </el-header>
    <el-main class="de-main">
      <el-form
        ref="mRightForm"
        class="m-form"
        :model="formData"
        label-position="top"
        hide-required-asterisk
        @submit.native.prevent
      >
        <el-form-item
          prop="name"
          class="form-item"
          :rules="[requiredRule]"
        >
          <template #label>
            {{ $t('data_fill.form.form_name') }}
            <span
              style="color: red"
            >*</span>
          </template>
          <el-input
            v-model.trim="formData.name"
            required
            size="small"
            maxlength="50"
            show-word-limit
          />
        </el-form-item>

        <el-form-item
          prop="folder"
          class="form-item"
          :rules="[requiredRule]"
        >
          <template #label>
            {{ $t('data_fill.form.folder') }}
            <span
              style="color: red"
            >*</span>
          </template>
          <el-popover
            v-model="folderTreeShow"
            placement="bottom-start"
            popper-class="user-popper dataset-filed"
            width="552"
            trigger="click"
          >
            <el-tree
              ref="tree"
              :data="folders"
              node-key="id"
              class="de-tree"
              :expand-on-click-node="false"
              highlight-current
              :filter-node-method="filterNode"
              default-expand-all
              @node-click="nodeClick"
            >
              <span
                slot-scope="{ data }"
                class="custom-tree-node-dataset"
              >
                <span>
                  <svg-icon icon-class="scene" />
                </span>
                <span
                  style="
                    margin-left: 6px;
                    white-space: nowrap;
                    overflow: hidden;
                    text-overflow: ellipsis;
                  "
                  :title="data.name"
                >{{ data.name }}</span>
              </span>
            </el-tree>
            <el-select
              slot="reference"
              v-model="formData.folder"
              filterable
              popper-class="tree-select-dataset"
              style="width: 100%"
              :filter-method="filterMethod"
              :placeholder="$t('commons.please_select')"
              required
            >
              <el-option
                v-for="item in selectDatasets"
                :key="item.label"
                :label="item.label"
                :value="item.id"
              />
            </el-select>
          </el-popover>

        </el-form-item>

        <el-form-item
          prop="datasource"
          class="form-item"
          :rules="[requiredRule]"
        >
          <template #label>
            {{ $t('data_fill.form.datasource') }}
            <span
              style="color: red"
            >*</span>
          </template>
          <el-select
            v-model="formData.datasource"
            filterable
            size="small"
            style="width: 100%"
          >
            <el-option-group
              v-for="(x, $index) in datasourceList"
              :key="$index"
              :label="x.name"
            >
              <el-option
                v-for="d in x.options"
                :key="d.id"
                :value="d.id"
                :label="d.name"
              >{{ d.name }}
              </el-option>
            </el-option-group>
          </el-select>
        </el-form-item>

        <el-form-item
          prop="table"
          class="form-item"
          :rules="[requiredRule]"
        >
          <template #label>
            {{ $t('data_fill.form.table_name') }}
            <span
              style="color: red"
            >*</span>
          </template>
          <el-input
            v-model.trim="formData.table"
            required
            size="small"
            maxlength="50"
            show-word-limit
          />
        </el-form-item>

        <el-table
          :data="formData.forms"
          border
          stripe
          style="width: 100%"
        >
          <el-table-column
            :label="$t('data_fill.form.form_column')"
          >
            <template slot-scope="scope">
              {{ scope.row.settings.name }}
            </template>
          </el-table-column>
          <el-table-column>
            <template
              slot="header"
            >
              {{ $t('data_fill.form.column_name') }}
            </template>
            <template slot-scope="scope">
              <el-form-item
                v-if="scope.row.type !== 'dateRange'"
                :prop="'forms['+scope.$index+'].settings.mapping.columnName'"
                class="form-item no-margin-bottom"
                :rules="[requiredRule, duplicateRule]"
              >
                <el-input
                  v-model.trim="scope.row.settings.mapping.columnName"
                  :placeholder="$t('fu.search_bar.please_input')"
                  size="small"
                  maxlength="50"
                  show-word-limit
                  required
                />
              </el-form-item>
              <template v-else>
                <el-form-item
                  :prop="'forms['+scope.$index+'].settings.mapping.columnName1'"
                  class="form-item no-margin-bottom"
                  :rules="[requiredRule, duplicateRule]"
                >
                  <el-input
                    v-model.trim="scope.row.settings.mapping.columnName1"
                    :placeholder="$t('data_fill.form.please_insert_start')"
                    size="small"
                    maxlength="50"
                    show-word-limit
                    required
                  />
                </el-form-item>
                <el-form-item
                  :prop="'forms['+scope.$index+'].settings.mapping.columnName2'"
                  class="form-item no-margin-bottom"
                  :rules="[requiredRule, duplicateRule]"
                >
                  <el-input
                    v-model.trim="scope.row.settings.mapping.columnName2"
                    :placeholder="$t('data_fill.form.please_insert_end')"
                    size="small"
                    maxlength="50"
                    show-word-limit
                    required
                  />
                </el-form-item>
              </template>
            </template>
          </el-table-column>
          <el-table-column
            :label="$t('data_fill.form.column_type')"
          >
            <template slot-scope="scope">
              <el-form-item
                :prop="'forms['+scope.$index+'].settings.mapping.type'"
                class="form-item no-margin-bottom"
                :rules="[requiredRule]"
              >
                <el-select
                  v-model="scope.row.settings.mapping.type"
                  :placeholder="$t('data_fill.form.please_select')"
                  size="small"
                  required
                  style="width: 100%"
                >
                  <el-option
                    v-for="o in scope.row.settings.mapping.typeOptions"
                    :key="o.value"
                    :value="o.value"
                    :label="o.label"
                  />

                </el-select>
              </el-form-item>
            </template>
          </el-table-column>
        </el-table>

        <div style="display: flex">
          <el-form-item
            prop="createIndex"
            class="form-item no-margin-bottom"
          >
            <el-checkbox
              v-model="formData.createIndex"
              :label="$t('data_fill.form.create_index')"
              size="small"
            />
          </el-form-item>

          <el-button
            v-if="formData.createIndex"
            type="text"
            style="margin-left: 20px"
            @click="addIndex"
          >+ {{ $t('data_fill.form.add_index') }}
          </el-button>
        </div>

        <el-table
          v-if="formData.createIndex"
          :data="formData.tableIndexes"
          border
          stripe
          style="width: 100%"
        >
          <el-table-column
            :label="$t('data_fill.form.index_name')"
            width="300"
          >
            <template slot-scope="scope">
              <el-form-item
                :prop="'tableIndexes['+scope.$index+'].name'"
                class="form-item"
                :rules="[requiredRule, duplicateIndexRule]"
              >
                <el-input
                  v-model="scope.row.name"
                  :placeholder="$t('fu.search_bar.please_input')"
                  size="small"
                  maxlength="50"
                  show-word-limit
                  required
                />
              </el-form-item>
            </template>
          </el-table-column>

          <el-table-column>
            <template
              slot="header"
              slot-scope="scope"
            >
              {{ $t('data_fill.form.index_column') }}
              <el-tooltip
                class="item"
                effect="dark"
                placement="bottom"
              >
                <div
                  slot="content"
                  v-html="$t('data_fill.form.create_index_hint')"
                />
                <i
                  class="el-icon-info"
                  style="cursor: pointer;"
                />
              </el-tooltip>
            </template>
            <template slot-scope="scope">
              <div
                v-for="(indexRow, $index) in scope.row.columns"
                :key="$index"
                style="display: flex; flex-direction: row; align-items: center;"
              >
                <el-form-item
                  :prop="'tableIndexes['+scope.$index+'].columns['+$index+'].column'"
                  class="form-item no-margin-bottom"
                  :rules="[requiredRule, invalidColumnRule, duplicateIndexColumnRule]"
                  style="flex: 1"
                >
                  <el-select
                    v-model="indexRow.column"
                    :placeholder="$t('data_fill.form.please_select')"
                    size="small"
                    required
                    style="width: 100%"
                  >
                    <el-option
                      v-for="(x, $index) in columnsList"
                      :key="$index"
                      :value="x"
                      :label="x"
                    >{{ x }}
                    </el-option>
                  </el-select>
                </el-form-item>

                <el-form-item
                  :prop="'tableIndexes['+scope.$index+'].columns['+$index+'].order'"
                  class="form-item no-margin-bottom"
                  :rules="[requiredRule]"
                >
                  <el-select
                    v-model="indexRow.order"
                    :placeholder="$t('data_fill.form.please_select')"
                    size="small"
                    required
                    style="width: 100%"
                  >
                    <el-option
                      value="asc"
                      :label="$t('data_fill.form.order_asc')"
                    />
                    <el-option
                      value="none"
                      :label="$t('data_fill.form.order_none')"
                    />
                    <el-option
                      value="desc"
                      :label="$t('data_fill.form.order_desc')"
                    />
                  </el-select>
                </el-form-item>
                <div
                  v-if="scope.row.columns.length > 1"
                  class="btn-item"
                  @click="removeIndexColumn(scope.row.columns, $index)"
                >
                  <svg-icon icon-class="icon_delete-trash_outlined" />
                </div>
              </div>
              <el-button
                v-if="scope.row.columns.length < 5"
                type="text"
                @click="addColumn(scope.row.columns)"
              >+ {{ $t('data_fill.form.add_column') }}
              </el-button>
            </template>
          </el-table-column>

          <el-table-column width="50">
            <template slot-scope="scope">
              <div
                class="btn-item"
                @click="removeIndex(scope.$index)"
              >
                <svg-icon icon-class="icon_delete-trash_outlined" />
              </div>
            </template>
          </el-table-column>
        </el-table>

      </el-form>

    </el-main>
    <el-footer class="de-footer">
      <el-button @click="closeSave">{{ $t("commons.cancel") }}</el-button>
      <el-button
        type="primary"
        @click="doSave"
      >{{ $t("commons.confirm") }}
      </el-button>
    </el-footer>
  </el-container>
</template>

<style  lang="scss" scoped>
.DataFillingFormSave {

  height: 100%;

  ::v-deep .el-form-item__error {
    position: relative;
  }

  .de-header {
    height: 56px !important;
    padding: 0px !important;
    border-bottom: 1px solid #E6E6E6;
    background-color: var(--SiderBG, white);

    display: flex;
    flex-direction: row;
    align-items: center;
    justify-content: space-between
  }

  .de-footer {
    display: flex;
    flex-direction: row;
    align-items: center;
    justify-content: flex-end
  }

  .panel-info-area {
    padding-left: 20px;
  }

  .de-main {
    display: flex;
    align-items: center;
    flex-direction: column;

    .m-form {
      width: 80%;
    }
  }

  .no-margin-bottom {
    margin-bottom: 0;
  }

  .btn-item {
    display: flex;
    flex-direction: row;
    align-items: center;
    justify-content: center;

    width: 24px;
    height: 24px;

    margin-left: 8px;

    border-radius: 4px;

    cursor: pointer;
  }

  .btn-item:first-child {
    margin-left: unset;
  }

  .btn-item:hover {
    background: rgba(31, 35, 41, 0.1);
  }

}
.dataset-filed {
  height: 400px;
  overflow-y: auto;
}

.tree-select-dataset {
  display: none;
}
</style>
