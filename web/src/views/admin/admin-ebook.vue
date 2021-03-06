<template>


  <a-layout>
    <a-layout-content
            :style="{ background: '#fff', padding: '24px', margin: 0, minHeight: '280px' }"
    >

      <p>
        <a-form layout="inline" :model="searchParams">
          <a-form-item>
            <a-input v-model:value="searchParams.name" placeholder="名称"/>
          </a-form-item>
          <a-form-item>
            <a-button type="primary" @click="handleQuery({page:1, size: pagination.pageSize, name: searchParams.name})">
              查询
            </a-button>
          </a-form-item>
          <a-form-item>
            <a-button type="primary" @click="add">
              新增
            </a-button>
          </a-form-item>
        </a-form>
      </p>
      <a-table
              :columns="columns"
              :row-key="record => record.id"
              :data-source="ebooks"
              :pagination="pagination"
              :loading="loading"
              @change="handleTableChange"
      >
        <!--自定义渲染封面列，用来展示显示图片-->
        <template #cover="{ text: cover }">
          <img v-if="cover" :src="cover" alt="avatar" />
        </template>

        <template v-slot:category="{ text, record }">
          <span>{{ getCategoryName(record.category1Id) }} / {{ getCategoryName(record.category2Id) }}</span>
        </template>

        <template v-slot:action="{ text, record }">
          <a-space size="small">
            <router-link :to="'/admin/doc?ebookId=' + record.id">
              <a-button type="primary">
                文档管理
              </a-button>
            </router-link>
            <a-button type="primary" @click="edit(record)">
              编辑
            </a-button>
            <a-popconfirm
                    title="删除后不可恢复，确认删除?"
                    ok-text="是"
                    cancel-text="否"
                    @confirm="del(record.id)"
            >
              <a-button type="danger">
                删除
              </a-button>
            </a-popconfirm>
          </a-space>
        </template>
      </a-table>

    </a-layout-content>
  </a-layout>

  <!-- Vue3的template下可以放置多个标签，Vue2则不支持 -->
  <a-modal
          title="电子书表单"
          v-model:visible="modalVisible"
          :confirm-loading="modalLoading"
          @ok="handleModalOk"
  >
    <!-- 模态框内的表单，用以编辑 -->
    <a-form :model="ebook" :label-col="{ span: 6 }" :wrapper-col="{ span: 18 }">
      <a-form-item label="封面">
        <a-input v-model:value="ebook.cover" />
      </a-form-item>
      <a-form-item label="名称">
        <a-input v-model:value="ebook.name" />
      </a-form-item>
      <a-form-item label="分类">
        <a-cascader
                v-model:value="categoryIds"
                :field-names="{ label: 'name', value: 'id', children: 'children' }"
                :options="level1"
        />
      </a-form-item>
<!--      <a-form-item label="分类1">-->
<!--        <a-input v-model:value="ebook.category1Id" />-->
<!--      </a-form-item>-->
<!--      <a-form-item label="分类2">-->
<!--        <a-input v-model:value="ebook.category2Id" />-->
<!--      </a-form-item>-->
      <a-form-item label="描述">
        <a-input v-model:value="ebook.description" type="textarea" />
      </a-form-item>
    </a-form>
  </a-modal>

</template>

<script lang="ts">
  import { defineComponent, onMounted, ref } from 'vue';
  import { message } from 'ant-design-vue';
  import axios from 'axios';
  import {Tool} from "@/util/tool";

  export default defineComponent({
    name: 'AdminEbook',
    setup: function () {
      const ebooks = ref();
      const pagination = ref({
        current: 1,
        pageSize: 4,
        total: 0
      });
      const loading = ref(false);

      const columns = [
        {
          title: '封面',
          dataIndex: 'cover',
          slots: {customRender: 'cover'}
        },
        {
          title: '名称',
          dataIndex: 'name'
        },
        {
          title: '分类',
          slots: { customRender: 'category' }
        },
        {
          title: '文档数',
          dataIndex: 'docCount'
        },
        {
          title: '阅读数',
          dataIndex: 'viewCount'
        },
        {
          title: '点赞数',
          dataIndex: 'voteCount'
        },
        {
          title: 'Action',
          key: 'action',
          slots: {customRender: 'action'}
        }
      ];

      const searchParams = ref({});

      const handleQuery = (params: any) => {
        loading.value = true;
        // GET请求需要传入params参数，POST请求则无此限制
        axios.get('/ebook/get', {
          params: {
            page: params.page,
            size: params.size,
            name: params.name,
          }
        }).then((response) => {
                  loading.value = false;
                  const data = response.data;
                  ebooks.value = data.content.records;
                  // 重置分页按钮
                  pagination.value.current = params.page;
                  pagination.value.total = data.content.total;
        });
      };

      const handleTableChange = (pagination : any) => {
        handleQuery({
          page: pagination.current,
          size: pagination.pageSize
        });
      };

      // 编辑对话框用到的变量
      const modalVisible = ref(false);
      const modalLoading = ref(false);
      // 用以存储当前行的数据
      const ebook = ref();
      const categoryIds = ref();
      const handleModalOk = () => {
        modalLoading.value = true;
        ebook.value.category1Id = categoryIds.value[0];
        ebook.value.category2Id = categoryIds.value[1];
        axios.post('/ebook/save', ebook.value)
             .then((response) => {
               modalLoading.value = false;
               if(response.data.success){
                 modalVisible.value = false;

                 handleQuery({
                   page: pagination.value.current,
                   size: pagination.value.pageSize
                 });
               } else {
                 message.error(response.data.message);
               }
             });

      };

      /**
       * 编辑
       */
      const edit = (record: any) => {
        modalVisible.value = true;
        console.log('record: ', record);
        ebook.value = Tool.copy(record);
        console.log("edit:  ", ebook.value);
        categoryIds.value = [ebook.value.category1Id, ebook.value.category2Id]
      };

      /**
       * 新增
       */
      const add = () => {
        modalVisible.value = true;
        modalLoading.value = false;
        ebook.value = {};
      };

      /**
       * 删除
       */
      const del = (id : string) => {
        axios.delete('/ebook/delete/' + id)
             .then((response) => {
               if(response.data.success){
                 handleQuery({
                   page: pagination.value.current,
                   size: pagination.value.pageSize
                 });
               }
        });
      };


      /**
       * 获取所有分类信息
       */
      const level1 = ref();
      let categorys: any;
      const handleQueryCategory = () => {
        // 如果不清空现有数据，编辑保存重新加载数据后，列表显示的还是编辑前的数据
        ebooks.value = [];
        axios.get('/category/all').then((response) => {
          if(response.data.success){

            handleQuery({
              page: 1,
              size: pagination.value.pageSize
            });

            level1.value= [];
            categorys = response.data.content;
            console.log("原始分类数据: ", categorys);

            level1.value = Tool.array2Tree(response.data.content,0);
            console.log("树形分类数据：", level1.value);
          } else{
            message.error("分类信息加载失败！");
          }
        });
      };

      /**
       * 根据id从分类信息中获取分类名
       * @param cid 分类id
       */
      const getCategoryName = (cid: number) => {
        console.log(cid);
        let result = "";

        if(Tool.isNotEmpty(categorys)){

          categorys.forEach((item: any) => {
            if (item.id === cid) {
              // return item.name; // 注意，这里直接return不起作用
              result = item.name;
              console.log('result: ',result);
            }
          });
          return result;
        }

      };

      onMounted(() => {
        handleQueryCategory();
        // 页面刚加载，应该查询第一页的数据

      });

      return {
        ebooks,
        pagination,
        columns,
        loading,
        handleTableChange,

        edit,
        add,
        del,

        modalVisible,
        modalLoading,
        handleModalOk,
        ebook,

        searchParams,
        handleQuery,

        categoryIds,
        level1,
        handleQueryCategory,
        getCategoryName,
      };

    }
  });
</script>
