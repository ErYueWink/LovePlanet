# Table组件封装

**组件封装的意义：**

1. **代码复用性：**封装组件可以使代码重复利用，避免重复编写相同的代码，提高开发效率。
2. **模块化开发：**封装组件可以使代码更加模块化，使得代码结构更加清晰，便于维护和管理。
3. **代码维护性：**封装组件可以使得代码更加简洁，便于维护和更新。
4. **组件复用性：**封装组件可以使组件在不同项目中重复使用，减少重复开发。
5. **提高可维护性：**封装组件可以使得代码更加可维护，减少维护难度。
6. **提高可维护性：**封装组件可以使得代码更加可维护，减少维护难度。
7. **提高可测试性：**封装组件可以使得代码更加容易测试

封装组件可以使得前端项目更加高效，更容易维护和扩展。

```vue
<template>
  <div class="gvb_table">
    <div class="gvb_table_head">
        <div class="action_create">
          <a-button type="primary" v-if="!props.noAdd" @click="add">创建用户</a-button>
        </div>
        <div class="action_group" v-if="!noActionGroup">
          <a-select placeholder="操作" allow-clear :options="actionOptions" v-model="actionValue"></a-select>
          <a-popconfirm content="确定要进行此操作吗?" v-if="!props.noConfirm"  @ok="actionMethod">
            <a-button status="danger" type="primary" v-if="actionValue !== undefined & actionValue !== ''">执行</a-button>
          </a-popconfirm>
          <a-button status="danger" v-else type="primary" v-if="actionValue !== undefined & actionValue !== ''" @click="actionMethod">执行</a-button>
        </div>
        <div class="action_search">
          <a-input-search @keydown.enter="search" :placeholder="props.searchPlackHolder" v-model="params.key" @search="search"></a-input-search>
        </div>
      <div class="action_filter">
        <a-select v-for="item in filterGroup" :placeholder="item.label" :options="item.options as commonOptionType[]"
                  @change="filterChange(item,$event)" allow-clear />
      </div>
        <div class="action_other_search"></div>
        <div class="action_slot"></div>
        <div class="action_flush">
          <a-button @click="refresh">
            <IconRefresh/>
          </a-button>
        </div>
    </div>
      <a-spin class="gvb_table_data" :loading="isLoading" tip="加载中...">
        <div class="gvb_table_source">
          <a-table :row-key="rowKey" :columns="props.columns" :data="data.list"
                   :row-selection="props.noCheck ? undefined : rowSelection"
                   v-model:selectedKeys="selectedKeys" :pagination="false">

            <!--          自定义列单元格渲染-->
            <template #columns>
              <template  v-for="item in props.columns">
                <!--                有render函数，优先执行render函数-->
                <a-table-column v-if="item.render" :title="item.title as string">
                  <template #cell="data">
                    <component :is="item.render(data) as Component"/>
                  </template>
                </a-table-column>
                <!--              没有slotName执行dataIndex-->
                <a-table-column  v-if="!item.slotName" :title="item.title as string" :data-index="item.dataIndex"></a-table-column>
                <!--              有slotName-->
                <a-table-column  v-else :title="item.title as string">
                  <!--                操作-->
                  <template #cell="{record}" v-if="item.slotName === 'action'">
                    <div class="gvb_table_action">
                      <slot name="action_left"></slot>
                      <a-button type="primary" v-if="!props.noEdit" @click="edit(record)">编辑</a-button>
                      <slot name="action_middle"></slot>
                      <a-popconfirm content="确定要删除吗?" @ok="remove(record)">
                        <a-button status="danger" v-if="!props.noRemove" type="primary" >删除</a-button>
                      </a-popconfirm>
                      <slot name="action_right"></slot>
                    </div>
                  </template>
                  <!--                日期格式化-->
                  <template #cell="{record}" v-if="item.slotName === 'created_at'">
                    <span>{{dateFormat(record.created_at)}}</span>
                  </template>
                  <!--                头像-->
                  <template #cell="{record}" v-if="item.slotName === 'avatar'">
                    <slot :name="item.slotName" :record="record" />
                  </template>
                </a-table-column>
              </template>
            </template>
          </a-table>
        </div>
        <div class="gvb_table_page">
          <a-pagination  v-model:current="params.page" :default-page-size="params.limit as number"
                         :total="data.count"
                         @change="pageChange"
                         show-jumper show-total/>
        </div>
      </a-spin>

  </div>
</template>

<script setup lang="ts">
import {IconRefresh} from "@arco-design/web-vue/es/icon";
import {type Component, reactive, ref} from "vue";
import {Message, type TableData, type TableRowSelection} from "@arco-design/web-vue";
import {type baseResponse, defaultOptionList, type paramsType} from "@/api";
import {TableColumnData} from "@arco-design/web-vue";
import {dateFormat} from "@/utils/date";
import {defaultDelete} from "@/api";
import type {commonOptionType} from "@/types";

export interface optionType{
  label:string
  value?:string | number
  callback?:  (idList: (string | number)[]) => Promise<boolean>
}

type filterFunc = (params?: paramsType) => Promise<baseResponse<commonOptionType[]>>

export interface actionFilterGroupType{
  label:string,
  value: number | string,
  column: string,
  source: commonOptionType[] | string | filterFunc
  options?: commonOptionType[]
}

interface Props {
  url:(params:paramsType)=>Promise<baseResponse<any>>
  columns: TableColumnData[],
  limit?:number,
  rowKey?:string,
  defaultDelete?:boolean, // 是否启用默认删除 default:false
  noActionGroup?:boolean, // 是否显示操作组
  actionGroup?:optionType[],
  noCheck?:boolean, // 不能选择的 default:false
  actionFilterGroup?: actionFilterGroupType[], // 过滤组
  noConfirm?:boolean, // 是否显示气泡弹出框 default:false
  noAdd?:boolean,
  noEdit?:boolean,
  noRemove?:boolean,
  searchPlackHolder?:string, // 搜索框提示信息
  defaultParams?:paramsType&any, // 默认查询参数

}

const props = defineProps<Props>()

export type RecordType <T> = T &{

}

// const emits = defineEmits(["add","edit","remove"])
const emits = defineEmits<{
  (e:'add') : void,
  (e:'edit',record:RecordType<any>) : void,
  (e:'remove',idList:(number | string)[]) :void
}>()
// 设置页数默认值
const {
  limit = 10,
  rowKey = "id"
} = props

const selectedKeys = ref<number[] | string[]>([]);

const rowSelection = reactive<TableRowSelection>({
  type: 'checkbox',
  showCheckedAll: true,
  onlyCurrent: false,
});

// 列表数据
const data = reactive({
  list:[],
  count:0
})

// 查询参数
const params = reactive<paramsType>({
  page:1,
  limit:limit,
  key:""
})

const isLoading = ref<boolean>(false)

async function getList(p?: optionType & any){
  if (p){
    Object.assign(params,p)
  }
  isLoading.value = true
  let res = await  props.url(params)
  isLoading.value = false
  data.list = res.data.list
  data.count = res.data.count
}
getList(props.defaultParams)

function pageChange(){
  getList()
}

function search(){
  params.page = 1
  getList()
}

function refresh(){
    Message.success("刷新成功")
    getList()
}

const add = () =>{
  emits("add")
}

const  edit = (record: RecordType<any>) =>{
  emits("edit",record)
}



const urlRegex = /return useAxios.get\("(.*?)",.*?\)/

const remove = async (record: RecordType<any>) =>{
  let id = record["id"]
  // 启用默认删除
  if (props.defaultDelete){
    // 正则匹配
    let resRegex = urlRegex.exec(props.url.toString())
    console.log(resRegex)
    if (resRegex === null || resRegex.length !== 2){ // 正则匹配失败
      return
    }
    // 匹配成功，格式正确，默认删除
    let res = await defaultDelete(resRegex[1],[id])
    console.log(res)
    if (res.code){
      Message.error(res.msg)
      return
    }
    Message.success(res.msg)
    getList()
    return
  }
  emits("remove",[id])
  return
}



// 分组
const actionOptions = ref<optionType[]>([
  {label:"批量删除",value:0}
])

// TODO 初始化操作组
const initActionGroup = ()=>{
  if (!props.actionGroup) return;
  for (let i = 0; i < props.actionGroup.length; i++) {
    actionOptions.value.push({
      label:props.actionGroup[i].label,
      value:i + 1,
      callback:props.actionGroup[i].callback
    })
  }
}

initActionGroup()
const actionValue = ref<string | number | undefined>(undefined)

const removeUserById = async (idList: number[] | string[]) =>{
  if (props.defaultDelete){
    // 正则匹配
    let resRegex = urlRegex.exec(props.url.toString())
    console.log(resRegex)
    if (resRegex === null || resRegex.length !== 2){ // 正则匹配失败
      return
    }
    // 匹配成功，格式正确，默认删除
    let res = await defaultDelete(resRegex[1],idList)
    console.log(res)
    if (res.code){
      Message.error(res.msg)
      return
    }
    Message.success(res.msg)
    getList()
    return
  }
  emits("remove",idList)
}

const actionMethod =  () =>{
  // 判断选中的是否为批量删除
  if (actionValue.value === 0){
    // 判断有没有传入ID
    if (selectedKeys.value.length === 0) {
      Message.warning("请选择要删除的项")
      return
    }
     removeUserById(selectedKeys.value)
    console.log(selectedKeys.value)
    return;
  }
  let action = actionOptions.value[actionValue.value]
   action.callback(selectedKeys.value).then(res =>{
     if (res){
       selectedKeys.value = []
       getList()
       return
     }
   })

}

// TODO 过滤组
const filterGroup = ref<actionFilterGroupType[]>([])

const filterChange = (item :any,val:any) =>{
  console.log(item,val)
  getList({[item.column]: val})
}

// 初始化过滤组
const initFilterGroup = async () =>{
  if (!props.actionFilterGroup) return
  for (let i = 0; i < props.actionFilterGroup.length; i++) {
    const item = props.actionFilterGroup[i]
    let option :commonOptionType[] = []
    switch (typeof item.source){
      case "function":
        let res1 = await (item.source as filterFunc)()
          option = res1.data
        break
      case "object":
        option = item.source
        break
      case "string":
        let res2 = await (defaultOptionList as filterFunc)(item.source)
        option = res2.data
        break
    }
    filterGroup.value.push({
      label:item.label,
      value:i,
      column:item.column,
      source:item.source,
      options:option
    })
  }
}
initFilterGroup()

defineExpose({
  getList
})

</script>


<style lang="scss">
.gvb_table{
  background-color: var(--color-bg-1);
  border-radius: 5px;
  .gvb_table_head{
    padding: 20px 20px 10px 20px;
    border-bottom: 1px solid var(--bg);
    display: flex;
    align-items: center;
    position: relative;
    >div{
      margin-right: 10px;
    }
    .action_flush{
      position: absolute;
      right: 20px;
      margin-right: 0;
      button{
        padding: 10px 10px;
      }
    }
    .action_group{
      display: flex;
      .arco-select-view-single{
        width: 120px;
      }
      button{
        margin-left: 10px;
      }
    }
    .action_filter{
      display: flex;
      .arco-select-view-single{
        width: 120px;
      }
      >.arco-select{
        margin-right: 15px;
      }
    }
  }
  .gvb_table_data{
    width: 100%;
    padding: 10px 20px 20px 20px;
    .gvb_table_source{
      .gvb_table_action{
        button{
          margin-right: 10px;
          &:last-child{
            margin-right: 0;
          }
        }

      }
    }
    .gvb_table_page{
      display: flex;
      justify-content: center;
      margin-top: 20px;
    }
  }
}
</style>
```

