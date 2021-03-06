<template>
  <div
    class="d2-crud d2-crud-x"
    :class="{'d2-crud-height100':options.height==='100%'}"
    v-loading="loading"
    :element-loading-text="getAttribute(loadingOptions,'text', null)"
    :element-loading-spinner="getAttribute(loadingOptions,'spinner', null)"
    :element-loading-background="getAttribute(loadingOptions,'background', null)"
  >
    <div class="d2-crud-header" v-if="$slots.header">
      <slot name="header"/>
    </div>
    <div class="d2-crud-body" v-if="options && options.hide === true">
      <slot name="body"/>
    </div>
    <div class="d2-crud-body" v-else >
      <el-table
        ref="elTable"
        :data="d2CrudData"
        v-bind="options"
        @current-change="handleCurrentChange"
        @select="handleSelect"
        @select-all="handleSelectAll"
        @selection-change="handleSelectionChange"
        @sort-change="handleSortChange"
        @cell-mouse-enter="handleCellMouseEnter"
        @cell-mouse-leave="handleCellMouseLeave"
        @cell-click="handleCellClick"
        @cell-dblclick="handleCellDblclick"
        @row-click="handleRowClick"
        @row-contextmenu="handleRowContextmenu"
        @row-dblclick="handleRowDblclick"
        @header-click="handleHeaderClick"
        @header-contextmenu="handleHeaderContextmenu"
      >
        <el-table-column
          v-if="selectionRow || selectionRow === ''"
          type="selection"
          :label="handleAttribute(selectionRow.title, '')"
          v-bind="selectionRow"
        >
        </el-table-column>
        <el-table-column
          v-if="expandRow || expandRow === ''"
          type="expand"
          :label="handleAttribute(expandRow.title, '')"
          v-bind="expandRow"
        >
          <template slot-scope="scope">
            <slot name="expandSlot" :row="scope.row"/>
          </template>
        </el-table-column>
        <el-table-column
          v-if="indexRow || indexRow === ''"
          type="index"
          :label="handleAttribute(indexRow.title, '')"
          v-bind="indexRow"
        >
        </el-table-column>
        <!-- 使用d2-column递归组件 -->
        <d2-column v-for="(item, index) in columns"
                   :key="index"
                   :item="item"
                   @cell-data-change="handleCellDataChange"
                   @cell-component-ready="handleCellComponentReady"
                   @cell-component-custom-event="handleCellComponentCustomEvent"
        >
            <template slot-scope="scope" :slot="item.key+'Slot'">
              <template v-if="item.rowSlot">
                <slot :name="item.key+'Slot'" :row="scope.row"/>
              </template>
            </template>
        </d2-column>

        <el-table-column
          v-if="rowHandle"
          :label="handleAttribute(rowHandle.columnHeader, '操作')"
          v-bind="rowHandle"
        >
          <template slot-scope="scope">
            <d2-button
              v-if="rowHandle.edit && handleRowHandleButtonShow(rowHandle.edit.show, scope.$index, scope.row)"
              :disabled="handleRowHandleButtonDisabled(rowHandle.edit.disabled, scope.$index, scope.row)"
              v-bind="rowHandle.edit"
              @click="handleEdit(scope.$index, scope.row)"
              :label="rowHandle.edit.text==null?undefined: handleAttribute(rowHandle.edit.text, '编辑')"
            />
            <d2-button
              v-if="rowHandle.remove && handleRowHandleButtonShow(rowHandle.remove.show, scope.$index, scope.row)"
              :type="handleAttribute(rowHandle.remove.type, 'danger')"
              :disabled="handleRowHandleButtonDisabled(rowHandle.remove.disabled, scope.$index, scope.row)"
              @click="handleRemove(scope.$index, scope.row)"
              :label="rowHandle.remove.text==null?undefined: handleAttribute(rowHandle.remove.text, '删除')"
              v-bind="rowHandle.remove"
            />
            <template
              v-for="(item, index) in handleAttribute(rowHandle.custom, [])"
            >
              <d2-button :key="index"
                         v-if="handleRowHandleButtonShow(item.show, scope.$index, scope.row)"
                         :disabled="handleRowHandleButtonDisabled(item.disabled, scope.$index, scope.row)"
                         v-bind="item"
                         @click="$emit(item.emit, {index: scope.$index, row: scope.row})"
                         :label="handleAttribute(item.text)"
              />
            </template>

          </template>

        </el-table-column>
      </el-table>
    </div>
    <div class="d2-crud-pagination" v-if="pagination">
      <el-pagination
        v-bind="pagination"
        @size-change="handlePaginationSizeChange"
        @current-change="handlePaginationCurrentChange"
        @prev-click="handlePaginationPrevClick"
        @next-click="handlePaginationNextClick"
      >
      </el-pagination>
    </div>

    <!-- 表单对话框 -->
    <el-dialog
      v-if="isDialogShow"
      :visible.sync="isDialogShow"
      :before-close="handleDialogCancel"
      v-bind="formOptions"
    >
      <template slot="title">
        {{formMode === 'edit' ? editTitle : addTitle}}
        <slot name="FormHeaderSlot" v-bind:mode="formMode" v-bind:data="formData" />
        <button v-if="formOptions.fullscreen!=null" type="button"  class="el-dialog__headerbtn" style="right:50px" @click="formOptions.fullscreen = !formOptions.fullscreen" ><i class="el-dialog__close el-icon el-icon-full-screen"></i></button>
      </template>
      <el-form
        ref="form"
        :model="formData"
        :rules="handleFormRulesMode()"
        :fullscreen="formOptions.fullscreen"
        v-bind="formOptions"
      >
        <el-row v-bind="formOptions" >
          <template v-for="(item,key, index) in formTemplateStorage" >
            <el-col :key="index"
              v-if="getTemplateComponentAttr(item,'show', true)"
              :span="getTemplateComponentAttr(item,'span', 24)"
              :offset="getTemplateComponentAttr(item,'offset', 0)"
            >
              <d2-form-item
                :template="item"
                :colKey="key"
                :formData="formData"
                @form-data-change="handleFormDataChange"
                @form-component-ready="handleFormComponentReady"
                @form-component-custom-event="handleFormComponentCustomEvent"
              >
                <template :slot="key+'FormSlot'">
                    <slot :name="key+'FormSlot'" :form="formData"/>
                </template>
                <template :slot="key+'HelperSlot'">
                    <slot :name="key+'HelperSlot'" :form="formData"/>
                </template>
              </d2-form-item>
            </el-col>
          </template>
        </el-row>

        <el-collapse v-if="formTemplateGroupStorage" v-model="formGroupsActive" :accordion="formTemplateGroupStorage.accordion"    >
          <el-collapse-item v-for="(group,groupKey) in formTemplateGroupStorage.groups" :name="groupKey" :key="groupKey" :disabled="group.disabled" >
            <template slot="title" >
              <h3 v-if="!$scopedSlots[groupKey+'GroupTitleSlot']" class="group-title" ><i v-if="group.icon" class="header-icon" :class="group.icon"/> {{group.title}}</h3>
              <slot :name="groupKey+'GroupTitleSlot'" :groupKey="groupKey" :group="group"/>
            </template>
            <el-row v-bind="formOptions">
              <template v-for="(item,key, index) in group.columns" >
                <el-col :key="index"
                        v-if="getTemplateComponentAttr(item,'show', true)"
                        :span="getTemplateComponentAttr(item,'span', 24)"
                        :offset="getTemplateComponentAttr(item,'offset', 0)"
                >
                  <d2-form-item
                    :template="item"
                    :colKey="key"
                    :formData="formData"
                    @form-data-change="handleFormDataChange"
                    @form-component-ready="handleFormComponentReady"
                    @form-component-custom-event="handleFormComponentCustomEvent"
                  >
                    <template :slot="key+'FormSlot'">
                      <slot :name="key+'FormSlot'" :form="formData"/>
                    </template>
                    <template :slot="key+'HelperSlot'">
                      <slot :name="key+'HelperSlot'" :form="formData"/>
                    </template>
                  </d2-form-item>
                </el-col>
              </template>
            </el-row>
          </el-collapse-item>
        </el-collapse>

        <slot name="FormBodyAppendSlot" :mode="formMode" :form="formData"/>
      </el-form>
      <div slot="footer">
          <slot name="FormFooterSlot" :mode="formMode" :data="formData" />
        <el-button v-if="getAttribute(formOptions,'saveButtonShow', true)"
          :size="getAttribute(formOptions,'saveButtonSize', null)"
          :type="getAttribute(formOptions,'saveButtonType' ,null)"
          :icon="getAttribute(formOptions,'saveButtonIcon', null)"
          :disabld="getAttribute(formOptions,'saveButtonDisabled', false)"
          :loading="getAttribute(formOptions,'saveLoading', false)"
          @click="handleDialogSave"
        >
        {{ getAttribute(formOptions,'saveButtonText', '确定')}}
        </el-button>
      </div>
    </el-dialog>
  </div>
</template>

<script>
import base from './mixin/base'
import handleRow from './mixin/handleRow'
import data from './mixin/data'
import edit from './mixin/edit'
import add from './mixin/add'
import remove from './mixin/remove'
import dialog from './mixin/dialog'
import pagination from './mixin/pagination'
import exposeMethods from './mixin/exposeMethods.js'
import utils from './mixin/utils'
import D2Column from './components/d2-column'
import D2FormItem from './components/d2-form-item'
import D2Button from './components/d2-button/component'

export default {
  name: 'd2-crud',
  mixins: [
    base,
    data,
    handleRow,
    edit,
    add,
    remove,
    dialog,
    pagination,
    exposeMethods,
    utils
  ],
  components: {
    D2FormItem,
    D2Column,
    D2Button
  },
  methods: {
    handleFormDataChange (event) {
      this.$emit('form-data-change', event)
    },
    handleFormComponentReady (event) {
      this.$emit('form-component-ready', event)
    },
    handleFormComponentCustomEvent (event) {
      this.$emit('form-component-custom-event', event)
    },
    handleCellDataChange (column) {
      this.$emit('cell-data-change', column)
    },
    handleCellComponentReady (column) {
      this.$emit('cell-component-ready', column)
    },
    handleCellComponentCustomEvent (column) {
      this.$emit('cell-component-custom-event', column)
    },
    handleDialogOpened (event) {
      // TODO 暂时无效
      this.$emit('form-dialog-opened', { event: event, form: this.formData })
    }
  }
}
</script>

<style lang="scss">
.d2-crud-x {
  display: flex;
  flex-direction: column;
  .d2-crud-header {
    /*border-bottom: 1px dotted rgba(0, 0, 0, 0.2);*/
  }
  .d2-crud-body {
    padding: 15px 0;
    overflow: hidden;
    flex-grow: 1;
  }
  .d2-crud-pagination {
    padding: 15px 0;
  }
  .form-item-helper{
    display: block;
    margin-top: 5px;
    margin-bottom: 5px;
    color: #737373;
    line-height: normal;
  }
  .el-collapse-item__wrap {
    padding-top: 7px;
  }
  .el-collapse-item__content{
    padding-left:30px;
    padding-right:30px;
  }
  &.d2-crud-height100{
    height:100%;
    .d2-crud-body{
      height: 100%;
    }
  }

  .d2-button{
    &.is-thin{
      &.el-button--small, &.el-button--small.is-round {
        padding: 9px 9px;
      }
      &.el-button--mini, &.el-button--mini.is-round {
        padding: 7px 7px;
      }
    }
  }
  .d2-crud-pagination{
    padding:0px 0;
  }
}

.page-compact{
  .d2-container-full__body{
    padding:0px !important
  }
  .d2-crud-header{
    padding:10px;
    border-bottom:0px
  }
  .d2-crud .d2-crud-body{
    padding:0px;
  }
  .d2-crud-pagination{
    padding:10px 0;
  }
  .el-table--border {
    border-left: none;
  }
}

.d2-mb-2{margin-bottom: 2px}
.d2-mt-2{margin-top: 2px;}
.d2-mr-5{margin-right: 5px}
.d2-mr-10{margin-right: 10px}

</style>
