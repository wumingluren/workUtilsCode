#### vue3列表页封装
vue3 element-plus 基于若依vue3项目做的列表页的封装
只封装数据而不封装页面

##### useList.js 只有获取列表数据一项功能
列表基础数据在这里定义，具体接口请求使用时传递，
搜索参数也是传递 ref 传递过来
```js
import { onMounted, ref, unref, watch } from 'vue';
import { ElNotification, ElMessageBox } from 'element-plus';

export default function useList(listRequestFn, options = {}) {
  const { queryParams = ref() } = options;
  const data = reactive({
    loading: false,
    pageIndex: 1, // 当前页
    pageSize: 10, // 分页大小
    pageSizes: [10, 20, 40, 50, 100],
    dataSource: [], // 列表数据
    total: 0, // 总条数
  });

  const { loading, pageIndex, pageSize, pageSizes, dataSource, total } = toRefs(data);

  const loadData = () => {
    return new Promise(async (resolve, reject) => {
      loading.value = true;
      try {
        // console.log(queryParams.value, 'hook查看搜索参数', queryParams);

        const result = await listRequestFn({
          pageIndex: pageIndex.value,
          pageSize: pageSize.value,
          ...queryParams.value,
        });

        // console.log(result, '接口请求结果', queryParams);

        dataSource.value = result.data.list;
        total.value = result.data.total;

        // 执行成功钩子
        options?.requestSuccess?.();
        resolve({
          dataSource: result.data.list,
          total: result.data.total,
        });
      } catch (error) {
        console.log('请求出错了', error);

        // 执行失败钩子
        options?.requestError?.(error);
      } finally {
        loading.value = false;
      }
    });
  };

  // 监听分页数据改变
  watch([pageIndex, pageSize], () => {
    console.log(pageIndex, '监听分页数据改变', pageSize);
    loadData();
  });

  // 备份默认搜索参数
  let backupQueryParams = {};

  // 重置查询条件
  const reset = () => {
    pageIndex.value = 1;
    pageSize.value = 10;
    queryParams.value = JSON.parse(JSON.stringify(backupQueryParams));
  };

  onMounted(() => {
    backupQueryParams = JSON.parse(JSON.stringify(queryParams.value));
    // console.log('list hook onMounted 执行了', backupQueryParams);
  });

  return {
    loading,
    pageIndex,
    pageSize,
    pageSizes,
    dataSource,
    total,
    loadData,
    reset,
  };
}
```
useTableList.js 删除等列表相关小功能
本来应该单独拆开的 懒得拆了
```js
import { computed, ref, onMounted } from 'vue';
import { httpAction, deleteAction, postAction, getAction } from '@/api/action';
import { ElNotification, ElMessageBox } from 'element-plus';

function useDelete(url, value, options = {}) {
  // console.log(url, '调用删除', options);
  const { paramName = 'id', requestSuccess = undefined, requestError = undefined } = options;

  deleteAction(url, {
    [paramName]: value,
  })
    .then((res) => {
      console.log('删除成功');

      ElNotification({
        title: '成功',
        message: '删除成功',
        type: 'success',
        duration: 2000,
      });

      // 执行成功钩子
      options?.requestSuccess?.();
    })
    .catch((error) => {
      console.log('删除请求出错了', error);

      // 执行失败钩子
      options?.requestError?.(error);
    })
    .finally(() => {});
}

export { useDelete };

```
使用
```js
import { useDelete } from '@/hooks/useTableList';
import useList from '@/hooks/useList';

const data = reactive({
  queryParams: {
    dateRange: [],
    test: 1,
  },
  rules: {
    // menuName: [{ required: true, message: '菜单名称不能为空', trigger: 'blur' }],
  },
});

const { queryParams, rules } = toRefs(data);

function getList(data) {
  return postAction('/user/manage/v1.0.0/getAgentByPage', data);
}

function requestSuccess() {
  console.log('请求执行成功', dataSource.value);
}

function requestError(error) {
  console.log('请求执行失败', error);
}

const { loading, pageIndex, pageSize, pageSizes, dataSource, total, loadData, reset } = useList(
  getList,
  {
    queryParams,
    requestSuccess,
    requestError,
  },
);

onMounted(() => {
  handleQuery();
});

/** 搜索按钮 */
function handleQuery() {
  loadData();
  // getStatistics();
}

function resetQuery() {
  reset();
  handleQuery();
}

/* 删除按钮 */
function handleDelete(row) {
  ElMessageBox.confirm(`确认删除？`, `删除`, {
    confirmButtonText: '确认',
    cancelButtonText: '取消',
    type: 'warning',
  })
    .then(() => {
      useDelete('/account/availChange/v1.0.0/delApply', row?.id, {
        requestSuccess: () => {
          console.log('删除接口成功');
          handleQuery();
        },
        requestError: () => {
          console.log('删除接口失败');
        },
      });
    })
    .catch(() => {
      // catch error
    });
}
```
##### 单个页面多个列表  比如详情页
原本：
```js
const {
  loading: realNameLoading,
  pageIndex: realNamePageIndex,
  pageSize: realNamePageSize,
  pageSizes: realNamePageSizes,
  dataSource: realNameDataSource,
  total: realNameTotal,
  loadData: realNameLoadData,
} = useList(
  (data) => {
    return postAction('/xxxx', data);
  },
  {
    queryParams,
    requestSuccess,
    requestError,
  },
);

const {
  loading: phoneChangeLoading,
  pageIndex: phoneChangePageIndex,
  pageSize: phoneChangePageSize,
  pageSizes: phoneChangePageSizes,
  dataSource: phoneChangeDataSource,
  total: phoneChangeTotal,
  loadData: phoneChangeLoadData,
} = useList(
  (data) => {
    return postAction('/xxx', data);
  },
  {
    queryParams,
    requestSuccess,
    requestError,
  },
);
```
一个一个去重命名也太麻烦了
修改一下 主要是view部分手动加上.value
一开始因为这个报错走了一些弯路
```js
<el-card class="box-card" style="margin-top: 20px">
  <template #header>
    <div class="">
      <span>实名认证记录</span>
    </div>
  </template>
  <el-table
    v-loading="realNameData.loading.value"
    :data="realNameData.dataSource.value"
    row-key="id"
    :border="true"
  >
    <el-table-column
      prop="name"
      label="姓名"
      align="center"
      :show-overflow-tooltip="true"
    ></el-table-column>
    <el-table-column prop="updateTime" label="修改时间"></el-table-column>
  </el-table>

  <pagination
    :total="realNameData.total.value"
    v-model:page="realNameData.pageIndex.value"
    v-model:limit="realNameData.pageSize.value"
    :pageSizes="realNameData.pageSizes.value"
  />
</el-card>
```
js部分
```js
const realNameData = useList(
  (data) => {
    return postAction('/xxx', data);
  },
  {
    queryParams,
    requestSuccess: () => {
      console.log('列表接口成功');
    },
    requestError: () => {
      console.log('列表接口失败');
    },
  },
);
```
