<template>
  <div class="table-wrap">
    <h2
      v-if="pageTitle"
      class="title">
      {{ pageTitle }}
    </h2>
    <div class="content">
      <div class="top">
        <div
          v-for="(column,index) in showQueryColumns"
          :key="index"
          class="selContainer">
          <!-- 下拉框 -->
          <span class="label">
            {{ column.name }}：
          </span>
          <el-select
            v-if="column.queryType=='select'"
            v-model="queryParams[column.id]"
            :disabled="column.support.query&&column.support.query.disabled"
            style="width:102px"
            filterable
            size="large"
            :placeholder="(column.support.query?
              column.support.query.placeholder:column.placeholder)||'请选择'"
            :clearable="true"
            @change="(val)=>{handleQueryChange(val, column.id)}">
            <el-option
              v-for="option in column.optionsForTable||column.options"
              :key="column.sourceFormat?option[column.sourceFormat.value]:option.id"
              :value="column.sourceFormat?option[column.sourceFormat.value]:option.id"
              :label="column.sourceFormat?option[column.sourceFormat.label]: option.name" />
          </el-select>
          <!-- input框 -->
          <el-input
            v-else
            v-model="queryParams[column.id]"
            size="large"
            :placeholder="(column.support.query?
              column.support.query.placeholder:column.placeholder)||'请输入内容'"
            @change="(val)=>{handleQueryChange(val, column.id)}" />
        </div>
        <div
          v-if="currentCount==initCount &&!noShowPagination"
          class="selContainer">
          <el-button
            v-if="!noShowSearchBtn"
            type="primary"
            size="large"
            @click="search(true)">
            查询
          </el-button>

          <template
            v-for="(btnConfig,index) in topBtnsConfig">
            <slot
              v-if="btnConfig.type==='slot'"
              :name="btnConfig.slotName" />


            <el-upload
              v-else-if="btnConfig.type=='upload'"
              :key="index"
              style="margin:0 8px"
              :show-file-list="false"
              :action="btnConfig.action"
              :on-error="handlerUploadErr"
              :data="btnConfig.data||{}"
              :on-success="()=>handlerUploadSucc(btnConfig.eventName)"
              :on-progress="()=>uploading=true">
              <el-button
                :loading="uploading"
                size="large"
                :plain="btnConfig.btnType&&btnConfig.btnType.isPlain"
                :type="getBtnType(btnConfig)">
                {{ btnConfig.name }}
              </el-button>
            </el-upload>

            <el-button
              v-else
              :key="index"
              :plain="btnConfig.btnType&&btnConfig.btnType.isPlain"
              :type="getBtnType(btnConfig)"
              size="large"
              @click="()=>handleTopBtnClick(btnConfig)">
              {{ btnConfig.name }}
            </el-button>
          </template>
        </div>
      </div>
      <div class="table">
        <el-table
          v-if="currentCount==initCount "
          v-loading="loading"
          :data="tableDataHandled"
          border
          size="medium"
          @selection-change="handleSelectionChange">
          <el-table-column
            v-if="showSelection"
            type="selection"
            align="center"
            width="55" />
          <el-table-column
            v-for="(one,index) of tableColumnHandled"
            v-show="tableColumnHandled.length"
            :key="index"
            :label="one.name"
            header-align="center"
            :align="one.align||'center'"
            :min-width="one.width||100">
            <template slot-scope="scope">
              <!-- 一个td显示多行 -->
              <div v-if="one.type===Array">
                <div
                  v-for="(item,idx) in scope.row[one.id]"
                  :key="idx"
                  style="float:left">
                  {{ `${idx+1}、${item}` }}
                </div>
              </div>
              <!-- 一个td只显示一个字段 -->
              <div
                v-else
                style="line-height:19px">
                {{ one.formatter? (Array.isArray(scope.row[one.id])? scope.row[one.id].map(val=>one.formatter(val)).join(',') :one.formatter(scope.row[one.id])):scope.row[one.id] }}
              </div>
            </template>
          </el-table-column>
          <el-table-column
            v-if="tableBtnsConfig.length"
            label="操作"
            :align="'center'"
            fixed="right">
            <template slot-scope="scope">
              <div style="text-align: center">
                <el-button
                  v-for="(oneConfig,index) in tableBtnsConfig"
                  v-show="typeof oneConfig.show==='function'?oneConfig.show(scope.row):oneConfig.show!==false"
                  :key="index"
                  type="text"
                  size="large"
                  :disabled="typeof oneConfig.disabled==='function'?oneConfig.disabled(scope.row):oneConfig.disabled"
                  @click="()=>handleTableBtnClick(oneConfig,scope.row)">
                  {{ oneConfig.name }}
                </el-button>
              </div>
            </template>
          </el-table-column>
        </el-table>
        <Pagination
          v-show="tableDataHandled.length &&!noShowPagination"
          :current-page="pageNo"
          :total="total"
          :page-sizes="pageSizes"
          :page-size="pageSize"
          @handleSizeChange="handleSizeChange"
          @handleCurrentChange="handleCurrentChange" />
      </div>
      <v-dialog
        ref="vDialog"
        :inputs="dataForDialog"
        @dialogClose="dialogClose"
        @edit="saveDialogEdit"
        @add="saveDialogAdd">
        <div
          v-for="(one,index) in slotItems"
          :key="index"
          :slot="one.slotName">
          <slot :name="one.slotName" />
        </div>
      </v-dialog>
    </div>
  </div>
</template>
<script>
  import Pagination from './Pagination.vue';
  import VDialog from './dialog.vue';
  import { formatDate } from '.utils';

  export default {
    components: { Pagination, VDialog },
    props: {
      // 表单字段名宽度，单位为px
      labelWidth: {
        type: Number,
        default: () => 90,
      },
      columns: {
        type: Array,
        required: true,
      },
      pageTitle: {
        type: String,
        default: () => '',
      },
      // 表格上方的按钮控制
      topBtnsConfig: {
        type: Array,
        default: () => [],
      },
      // 表格里面的按钮控制
      tableBtnsConfig: {
        type: Array,
        default: () => [],
      },
      // 查询条件变更是否马上查询
      searchOnChange: {
        type: Boolean,
        default: true,
      },
      // 父组件传入的动态的查询参数
      basicQueryForm: {
        type: Object,
        default: () => {},
      },

      basicAddForm: {
        type: Object,
        default: () => {},
      },
      basicEditForm: {
        type: Object,
        default: () => {},
      },


      getData: {
        type: Function,
        required: true,
      },

      noShowPagination: {
        default: () => false,
        type: Boolean,
      },

      noShowSearchBtn: {
        default: () => false,
        type: Boolean,
      },

      showSelection: {
        defalut: () => false,
        type: Boolean,
      },
      // 修改的时候不传给后端的字段
      notSendColumns: {
        type: Array,
        default: () => ['updateUser', 'updateTime', 'password', 'updatePassword', 'createTime'],
      },
      dailogWidth: {
        type: String,
        default: () => '33%',
      },
      formSize: {
        type: String,
        default: () => 'large',
      },
    },
    data() {
      return {
        dataForDialog: {
          show: false,
          mode: '', // 编辑或者新增，
          form: '', // 表单
          index: '', // 当前编辑行在表格的索引
          items: [], // 字段
          loading: false,
          formSize: this.formSize,
          dailogWidth: this.dailogWidth,
          labelWidth: this.labelWidth,
          basicAddForm: this.basicAddForm,
          basicEditForm: this.basicEditForm,
          notSendColumns: this.notSendColumns,
        },
        total: 0,
        pageSize: 20,
        pageSizes: [20, 50, 100],
        pageNo: 1,
        initCount: 0, // 需要远程获取option的数量
        currentCount: 0, // 已经初始化option的数量
        queryParams: {}, // 查询
        loading: false,
        tableDataHandled: [],
        uploading: false,
      };
    },
    computed: {
      showQueryColumns() {
        return this.columns.filter(one => {
          if (Array.isArray(one.support)) {
            return one.support.includes('query');
          } if (typeof one.support == 'object') {
            return !!one.support.query;
          }
          return false;
        });
      },
      tableColumnHandled() {
        return this.columns.filter(one => one.isShow !== false && !['title', 'slot'].includes(one.queryType));
      },
      slotItems() {
        return this.dataForDialog.items.filter(one => one.queryType == 'slot');
      },
    },
    watch: {
      basicQueryForm: {
        deep: true,
        handler() {
          this.search();
        },
      },
    },
    mounted() {
      this.initCount = this.columns.filter(one => one.source).length;
      this.columns.forEach(column => {
        // 初始化options
        if (column.source) {
          column.source().then(({ model }) => {
            if (!column.formatter) {
              this.generateFormatter(column);
            }
            this.currentCount++;
            this.$set(column, 'options', model); //
          }).catch(e => {
            this.$message.error(`获取${column.name}选项失败!`);
            console.log(e);
          });
        } else if ((column.options && column.options.length) || ['updateTime'].includes(column.id)) {
          if (!column.formatter) {
            this.generateFormatter(column);
          }
        }

        // 初始化查询表单
        if (column.support) {
          // 是数组时，是最基础的配置，复杂的配置时是一个对象
          if (Array.isArray(column.support)) {
            if (column.support.includes('query')) {
              this.$set(this.queryParams, column.id, '');
            }
          } else if (typeof column.support == 'object') {
            if (column.support.query) {
              this.$set(this.queryParams, column.id, '');
            }
            const modes = ['add', 'edit'];
            modes.forEach(mode => {
              if (column.support[mode] && column.support[mode].eventName) { // 编辑和新增，事件监听，并且向父组件抛出来
                this.$refs.vDialog.$on(column.support[mode].eventName, obj => {
                  this.$emit(column.support[mode].eventName, obj);
                });
              }
            });
          }
        }
      });
    },
    methods: {
      getBtnType(btnConfig, defaultType = 'primary') {
        return typeof btnConfig.btnType === 'object' ? btnConfig.btnType.type : (btnConfig.btnType || defaultType);
      },
      handlerUploadErr() {
        this.uploading = false;
        this.$message.error('文件上传失败');
      },
      handlerUploadSucc(eventName) {
        this.uploading = false;
        this.$emit(eventName);
      },
      handleTopBtnClick(config) {
        if (config.addConfig) {
          this.addOrEdit(config.addConfig.title);
        }
        if (config.eventName) {
          this.$emit(config.eventName, this.tableDataHandled, this.queryParams);
        }
      },
      // 表格按钮
      handleTableBtnClick(config, rowData) {
        // 组件内部完成弹框
        if (config.editConfig) {
          const index = this.tableDataHandled.indexOf(rowData);
          this.addOrEdit(config.editConfig.title, rowData, index);
        }
        // 需父组件处理
        if (config.eventName) {
          this.$emit(config.eventName, rowData, this.tableDataHandled);
        }
      },
      addOrEdit(title, rowData, index) {
        let mode = 'add';
        if (rowData) {
          mode = 'edit';
          this.dataForDialog.index = index;
          this.dataForDialog.form = rowData;
        }
        this.dataForDialog.mode = mode;
        this.dataForDialog.items = this.columns.filter(one => one.support && ((Array.isArray(one.support) && one.support.includes(mode)) || one.support[mode]));
        this.dataForDialog.title = title;
        this.$emit('beforeDialogOpen', this.dataForDialog, mode);
        this.dataForDialog.show = true;
      },

      // dialog编辑
      saveDialogEdit(form) {
        const config = this.tableBtnsConfig.find(one => one.editConfig);
        if (config && config.editConfig.handler) {
          config.editConfig.handler(form).then(({ model }) => {
            if (model && Object.prototype.toString.call(model) == '[object Object]'
            ) {
              this.$set(this.tableDataHandled, this.dataForDialog.index, model);
            } else {
              this.search();
            }
            this.dataForDialog.show = false;
            this.$message.success('保存成功');
          }).catch(e => {
            this.$message.error(e.message);
          });
        }
      },
      // dialog新增
      saveDialogAdd(form) {
        const config = this.topBtnsConfig.find(one => one.addConfig);
        if (config && config.addConfig.handler) {
          config.addConfig.handler(form).then(() => {
            this.dataForDialog.show = false;
            this.search();
            this.$message.success('保存成功');
          }).catch(e => {
            this.$message.error(e.message);
          });
        }
      },
      dialogClose() {
        this.$emit('dialogClose');
      },
      //          把选中的数据传入
      handleSelectionChange(rows) {
        this.$emit('selectChange', rows);
      },
      search(clearPage = false) {
        this.loading = true;
        clearPage && (this.pageNo = 1);
        const params = {
          pageSize: this.pageSize,
          pagination: this.pageNo,
        };
        const finalParams = Object.assign(params, this.queryParams, this.basicQueryForm);
        this.$emit('beforeQuery', finalParams);

        this.getData(finalParams)
          .then(res => {
            this.loading = false;
            const { model: { records, total } } = res;
            this.$emit('beforeAssignToTable', records);
            this.tableDataHandled = records;
            this.total = total;
          })
          .catch(e => {
            this.loading = false;
            console.log(e);
            this.$message.error(e.message);
          });
      },
      handleSizeChange(pageSize) {
        this.pageSize = pageSize;
        this.search();
      },
      //          分页跳转
      handleCurrentChange(currentPage) {
        this.pageNo = currentPage;
        this.search();
      },
      // 查询条件变化，没有点击搜索
      handleQueryChange(val, id) {
        this.$emit('queryChange', id, val);
        if (this.searchOnChange) {
          this.search();
        }
      },
      generateFormatter(column) {
        column.formatter = val => {
          // 统一格式化时间
          if (column.id === 'updateTime') {
            return formatDate(val);
          }
          const target = column.options.find(one => one[column.sourceFormat
            ? column.sourceFormat.value
            : 'id'] == val);
          return target ? target[
            column.sourceFormat
              ? column.sourceFormat.label
              : 'name'
          ] : val;
        };
      },
    },
  };
</script>
<style lang="scss" scoped>
  *{
    box-sizing: border-box;
  }
  .table-wrap{
    // padding: 0 10px 10px 10px;
    .title {
        display: inline-block;
        text-indent: 15px;
        /*border-left: 2px solid #88b7e0;*/
        margin-top: 15px;
        margin-bottom: 0px;
        margin-right: 8px;
        vertical-align: top;
    }
    .content {
        .top {
            float: left;
            padding: 6px;
            align-items: center;
            display: flex;
            .selContainer {
              align-items: center;
              display:flex;
              .label{
                color:#303133;
                font-size: 14px;
              }
              .el-select {
                  width: 180px;
              }

              .el-input {
                  width: 180px;

              }
              margin: 5px;
            }
        }

    }
  }
</style>
