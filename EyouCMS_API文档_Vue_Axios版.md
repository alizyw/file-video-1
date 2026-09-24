# EyouCMS API 接口文档（Vue + Axios 版）

> 基于 EyouCMS 标签化 API 接口体系，适用于 Vue2 / Vue3 项目的前后端分离开发。
> 官方文档参考：https://www.eyoucms.com/doc/api/

---

## 目录

1. [概述](#1-概述)
2. [环境准备](#2-环境准备)
3. [Axios 封装](#3-axios-封装)
4. [认证机制](#4-认证机制)
5. [通用调用规范](#5-通用调用规范)
6. [全局标签接口](#6-全局标签接口)
7. [列表标签接口](#7-列表标签接口)
8. [内容标签接口](#8-内容标签接口)
9. [用户相关接口](#9-用户相关接口)
10. [商城相关接口](#10-商城相关接口)
11. [完整调用示例](#11-完整调用示例)
12. [常见问题](#12-常见问题)

---

## 1. 概述

EyouCMS 采用**标签化 API 接口**设计，所有接口统一走一个入口地址，通过参数中的标签名（如 `apiList`、`apiGlobal`）区分不同的数据请求。

### 核心特点

- **统一入口**：所有 GET 类数据请求走同一个 API 地址
- **批量请求**：一次请求可同时调用多个标签接口，减少 HTTP 请求数
- **ekey 标识**：每个接口调用需指定 `ekey`（数字），返回数据按 ekey 分组
- **密钥认证**：后台配置 API 密钥，前端生成 token 进行鉴权

### 接口入口地址

```
# 数据查询接口（GET 类标签）
GET/POST  https://你的域名/index.php?m=api&c=index&a=index

# 表单提交 / 操作类接口（POST）
POST  https://你的域名/index.php?m=api&c=index&a=post
```

> 实际地址以你后台「基本信息 → 接口配置 → 小程序API」中显示的为准。

---

## 2. 环境准备

### 2.1 后台开启 API

1. 登录 EyouCMS 后台
2. 进入 **功能地图 → 基本信息 → 接口配置 → 小程序API**
3. 开启接口并设置 **API密钥（apikey）**
4. 保存后复制密钥值

### 2.2 前端项目配置

在 Vue 项目根目录创建环境变量文件：

```env
# .env.development
VITE_API_BASE_URL=https://your-domain.com
VITE_API_KEY=你的后台apikey值
VITE_API_TIMEOUT=15000
```

```env
# .env.production
VITE_API_BASE_URL=https://your-domain.com
VITE_API_KEY=你的后台apikey值
VITE_API_TIMEOUT=10000
```

### 2.3 安装依赖

```bash
npm install axios
# 或
yarn add axios
```

---

## 3. Axios 封装

### 3.1 创建请求实例

新建 `src/utils/request.js`：

```javascript
import axios from 'axios'
import { getApiKeyToken } from './auth'

// 创建 axios 实例
const service = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL,
  timeout: parseInt(import.meta.env.VITE_API_TIMEOUT) || 15000,
  headers: {
    'Content-Type': 'application/x-www-form-urlencoded;charset=UTF-8'
  }
})

// 请求拦截器
service.interceptors.request.use(
  config => {
    // 统一添加 API 认证 token
    const token = getApiKeyToken()
    if (token) {
      // GET 请求拼到 URL 参数，POST 请求加到 data
      if (config.method === 'get') {
        config.params = { ...config.params, apikey_token: token }
      } else {
        config.data = config.data || {}
        config.data.apikey_token = token
      }
    }
    return config
  },
  error => {
    console.error('请求错误:', error)
    return Promise.reject(error)
  }
)

// 响应拦截器
service.interceptors.response.use(
  response => {
    const res = response.data
    // EyouCMS 统一返回格式 { status: 1, msg: '', data: {} }
    if (res.status === 1 || res.code === 200) {
      return res
    }
    // 业务错误
    console.error('接口业务错误:', res.msg || res.message)
    return Promise.reject(new Error(res.msg || res.message || '请求失败'))
  },
  error => {
    console.error('网络错误:', error.message)
    return Promise.reject(error)
  }
)

export default service
```

### 3.2 封装 EyouCMS 专用请求方法

新建 `src/utils/eyouApi.js`：

```javascript
import request from './request'
import qs from 'qs'

/**
 * EyouCMS 标签化 API 统一请求方法
 * @param {Object} apiParams - 接口参数对象，格式 { apiList_1: 'ekey=1&typeid=2', apiGlobal_2: 'ekey=2' }
 * @returns {Promise}
 */
export function requestApi(apiParams) {
  return request({
    url: '/index.php',
    method: 'post',
    params: { m: 'api', c: 'index', a: 'index' },
    data: qs.stringify(apiParams)
  })
}

/**
 * EyouCMS 操作类接口（提交表单、点赞、收藏等）
 * @param {Object} data - 提交数据
 * @returns {Promise}
 */
export function requestPost(data) {
  return request({
    url: '/index.php',
    method: 'post',
    params: { m: 'api', c: 'index', a: 'post' },
    data: qs.stringify(data)
  })
}

/**
 * 单接口快捷调用
 * @param {string} apiName - 接口标签名，如 'apiList'
 * @param {string} paramStr - 参数串，如 'ekey=1&typeid=2&pagesize=10'
 * @param {number} ekey - 唯一标识，默认 1
 * @returns {Promise}
 */
export function callApi(apiName, paramStr, ekey = 1) {
  const key = `${apiName}_${ekey}`
  return requestApi({ [key]: paramStr })
}
```

---

## 4. 认证机制

### 4.1 apikey_token 生成规则

EyouCMS 的 API 认证基于后台配置的 `apikey`，前端需要生成 `apikey_token` 参数。

新建 `src/utils/auth.js`：

```javascript
import md5 from 'md5' // npm install md5

const API_KEY = import.meta.env.VITE_API_KEY

/**
 * 生成 apikey_token
 * 规则：md5(apikey + 当前时间戳的前6位)
 * 具体规则以你项目 app.js 中的实现为准，以下为官方小程序默认逻辑
 */
export function getApiKeyToken() {
  const timestamp = Math.floor(Date.now() / 1000).toString()
  // 取时间戳前6位
  const timePrefix = timestamp.substring(0, 6)
  return md5(API_KEY + timePrefix)
}

/**
 * 用户 token（登录后获取，存储在 localStorage）
 */
export function getUserToken() {
  return localStorage.getItem('eyou_user_token') || ''
}

export function setUserToken(token) {
  localStorage.setItem('eyou_user_token', token)
}

export function removeUserToken() {
  localStorage.removeItem('eyou_user_token')
}
```

> **注意**：`apikey_token` 的具体生成算法可能因 EyouCMS 版本不同而有差异。
> 请以官方小程序源码 `app.js` 中的 `apikey_token` 生成方法为准进行适配。

### 4.2 用户登录态认证

涉及用户操作的接口（如收藏、点赞、订单），还需在请求中携带用户 token：

```javascript
// 在请求拦截器中补充用户 token
service.interceptors.request.use(config => {
  const userToken = getUserToken()
  if (userToken) {
    if (config.method === 'get') {
      config.params.token = userToken
    } else {
      config.data.token = userToken
    }
  }
  return config
})
```

---

## 5. 通用调用规范

### 5.1 请求格式

所有数据查询接口统一使用 POST 方法，Content-Type 为 `application/x-www-form-urlencoded`。

**请求体格式**：

```
apiList_1=ekey=1&typeid=2&pagesize=10&apiGlobal_2=ekey=2
```

即：`接口名_ekey值=参数串`，多个接口用 `&` 连接。

### 5.2 响应格式

```json
{
  "status": 1,
  "msg": "success",
  "data": {
    "apiList": {
      "1": {
        "data": [...],
        "page": 1,
        "total": 100
      }
    },
    "apiGlobal": {
      "2": {
        "webname": "网站名称",
        "logo": "/uploads/logo.png"
      }
    }
  }
}
```

### 5.3 ekey 使用规则

- `ekey` 为**必填**参数，是一个数字标识
- 请求时参数名格式为 `接口名_ekey值`，如 `apiList_1`
- 返回数据中对应 `data.apiList[1]`
- **数字必须一致**：`ekey=1` 对应 `apiList_1` 和返回的 `apiList[1]`
- 一次请求中多个接口的 ekey 不能重复

### 5.4 批量请求示例

一次请求同时获取「网站全局配置」+「导航菜单」+「文章列表」：

```javascript
import { requestApi } from '@/utils/eyouApi'

async function loadPageData() {
  const res = await requestApi({
    apiGlobal_1: 'ekey=1',
    apiNavigation_2: 'ekey=2',
    apiList_3: 'ekey=3&typeid=1&pagesize=10'
  })

  const global = res.data.apiGlobal[1]
  const navigation = res.data.apiNavigation[2].data
  const articleList = res.data.apiList[3].data

  return { global, navigation, articleList }
}
```

---

## 6. 全局标签接口

### 6.1 apiGlobal — 全局配置变量

获取网站全局配置信息（网站名、Logo、SEO、联系方式等）。

**参数**：

| 参数 | 必填 | 说明 | 示例 |
|------|------|------|------|
| ekey | 是 | 唯一标识 | `ekey=1` |

**调用示例**：

```javascript
import { callApi } from '@/utils/eyouApi'

const res = await callApi('apiGlobal', 'ekey=1')
const siteConfig = res.data.apiGlobal[1]
// siteConfig.webname, siteConfig.logo, siteConfig.seo_keywords ...
```

---

### 6.2 apiNavigation — 导航菜单

获取网站导航栏目列表。

**参数**：

| 参数 | 必填 | 说明 | 示例 |
|------|------|------|------|
| ekey | 是 | 唯一标识 | `ekey=1` |

**调用示例**：

```javascript
const res = await callApi('apiNavigation', 'ekey=1')
const navList = res.data.apiNavigation[1].data
// navList 为树形结构，含 typedir, typename, children 等
```

---

### 6.3 apiChannel — 栏目列表

获取指定模型或全部栏目的列表。

**参数**：

| 参数 | 必填 | 说明 | 示例 |
|------|------|------|------|
| ekey | 是 | 唯一标识 | `ekey=1` |
| channelid | 否 | 频道ID（1=文章, 2=产品, 3=图集, 4=下载, 6=视频） | `channelid=1` |
| typeid | 否 | 指定栏目ID，获取子栏目 | `typeid=2` |

**调用示例**：

```javascript
const res = await callApi('apiChannel', 'ekey=1&channelid=1')
const channels = res.data.apiChannel[1].data
```

---

### 6.4 apiAdv — 广告列表

获取广告位列表。

**参数**：

| 参数 | 必填 | 说明 | 示例 |
|------|------|------|------|
| ekey | 是 | 唯一标识 | `ekey=1` |
| pid | 否 | 广告位ID | `pid=1` |

**调用示例**：

```javascript
const res = await callApi('apiAdv', 'ekey=1&pid=1')
const banners = res.data.apiAdv[1].data
```

---

### 6.5 apiAd — 单条广告

获取单条广告信息。

**参数**：

| 参数 | 必填 | 说明 | 示例 |
|------|------|------|------|
| ekey | 是 | 唯一标识 | `ekey=1` |
| aid | 是 | 广告ID | `aid=1` |

---

### 6.6 apiFlink — 友情链接

获取友情链接列表。

**参数**：

| 参数 | 必填 | 说明 | 示例 |
|------|------|------|------|
| ekey | 是 | 唯一标识 | `ekey=1` |
| typeid | 否 | 分类ID | `typeid=1` |

---

### 6.7 apiType — 栏目信息

获取单个栏目的详细信息。

**参数**：

| 参数 | 必填 | 说明 | 示例 |
|------|------|------|------|
| ekey | 是 | 唯一标识 | `ekey=1` |
| typeid | 是 | 栏目ID | `typeid=2` |

---

## 7. 列表标签接口

### 7.1 apiList — 文档列表（分页）

获取栏目文档列表，支持分页，是最常用的接口。

**参数**：

| 参数 | 必填 | 说明 | 示例 |
|------|------|------|------|
| ekey | 是 | 唯一标识 | `ekey=1` |
| typeid | 否 | 栏目ID，多个用逗号分隔（需同模型） | `typeid=1,2` |
| notypeid | 否 | 排除的栏目ID | `notypeid=3,4` |
| channelid | 否 | 频道ID，优先级高于 typeid | `channelid=1` |
| pagesize | 否 | 每页条数，默认 10 | `pagesize=15` |
| page | 否 | 当前页码 | `page=2` |
| titlelen | 否 | 标题截取长度 | `titlelen=30` |
| infolen | 否 | 简介截取长度 | `infolen=160` |
| orderby | 否 | 排序字段：hot/click/add_time/aid/sort_order | `orderby=add_time` |
| ordermode | 否 | 排序方式：desc/asc | `ordermode=desc` |
| flag | 否 | 自定义属性：c(推荐) j(跳转) | `flag=c` |
| noflag | 否 | 排除的属性 | `noflag=j` |
| keywords | 否 | 关键词搜索 | `keywords=新闻` |
| addfields | 否 | 附加自定义字段，逗号分隔 | `addfields=price,spec` |

**调用示例**：

```javascript
import { callApi } from '@/utils/eyouApi'

// 获取栏目ID=2的文章列表，每页10条，按发布时间倒序
const res = await callApi('apiList', 'ekey=1&typeid=2&pagesize=10&orderby=add_time&ordermode=desc')

const list = res.data.apiList[1].data      // 文章列表数组
const total = res.data.apiList[1].total    // 总条数
const page = res.data.apiList[1].page      // 当前页
```

**常用场景参数组合**：

```
# 最新文章
ekey=1&typeid=1&pagesize=10&orderby=add_time

# 热门文章
ekey=1&typeid=1&pagesize=10&orderby=click&ordermode=desc

# 推荐文章
ekey=1&typeid=1&flag=c&pagesize=10

# 关键词搜索
ekey=1&keywords=产品&pagesize=10

# 含自定义字段
ekey=1&typeid=3&pagesize=10&addfields=price,stock
```

---

### 7.2 apiArclist — 自由文档

获取自由文档列表，不带分页，适合取固定数量数据（如首页推荐区块）。

**参数**：与 apiList 基本一致，额外支持：

| 参数 | 必填 | 说明 | 示例 |
|------|------|------|------|
| row | 否 | 获取条数（替代 pagesize） | `row=6` |
| aid | 否 | 指定文档ID，多个逗号分隔 | `aid=1,2,3` |

**调用示例**：

```javascript
// 首页取6条推荐产品
const res = await callApi('apiArclist', 'ekey=1&typeid=3&row=6&flag=c')
const products = res.data.apiArclist[1].data
```

---

### 7.3 apiChannellist — 栏目分页列表

按栏目分组的分页列表，适合栏目聚合页。

**参数**：参考 apiList。

---

### 7.4 apiCommentlist — 评论分页

获取文档评论列表。

**参数**：

| 参数 | 必填 | 说明 | 示例 |
|------|------|------|------|
| ekey | 是 | 唯一标识 | `ekey=1` |
| aid | 是 | 文档ID | `aid=10` |
| pagesize | 否 | 每页条数 | `pagesize=10` |
| page | 否 | 页码 | `page=1` |

---

### 7.5 apiLikearticle — 相关文档

获取与指定文档相关的文章列表。

**参数**：

| 参数 | 必填 | 说明 | 示例 |
|------|------|------|------|
| ekey | 是 | 唯一标识 | `ekey=1` |
| aid | 是 | 当前文档ID | `aid=10` |
| row | 否 | 相关文章条数 | `row=5` |

---

## 8. 内容标签接口

### 8.1 apiCollect — 文档收藏

收藏 / 取消收藏文档（需登录）。

**参数（POST 操作接口）**：

| 参数 | 必填 | 说明 |
|------|------|------|
| aid | 是 | 文档ID |
| opt | 是 | 操作类型：add(收藏) / del(取消) |

**调用示例**：

```javascript
import { requestPost } from '@/utils/eyouApi'

await requestPost({
  apiCollect: '',
  aid: 10,
  opt: 'add'
})
```

---

### 8.2 apiLike — 文档点赞

点赞 / 取消点赞文档。

**参数**：

| 参数 | 必填 | 说明 |
|------|------|------|
| aid | 是 | 文档ID |
| opt | 是 | add / del |

---

### 8.3 apiPrenext — 上下篇

获取当前文档的上一篇和下一篇。

**参数**：

| 参数 | 必填 | 说明 | 示例 |
|------|------|------|------|
| ekey | 是 | 唯一标识 | `ekey=1` |
| aid | 是 | 当前文档ID | `aid=10` |
| typeid | 是 | 栏目ID | `typeid=2` |

**调用示例**：

```javascript
const res = await callApi('apiPrenext', 'ekey=1&aid=10&typeid=2')
const prenext = res.data.apiPrenext[1]
// prenext.prev (上一篇), prenext.next (下一篇)
```

---

### 8.4 apiForm — 自由表单

获取自由表单字段列表，用于动态渲染表单。

**参数**：

| 参数 | 必填 | 说明 | 示例 |
|------|------|------|------|
| ekey | 是 | 唯一标识 | `ekey=1` |
| formid | 是 | 表单ID | `formid=1` |

**调用示例**：

```javascript
const res = await callApi('apiForm', 'ekey=1&formid=1')
const formFields = res.data.apiForm[0] // 注意返回索引可能是 0
```

---

### 8.5 apiGuestbookform — 留言栏目

获取留言板表单字段。

**参数**：

| 参数 | 必填 | 说明 | 示例 |
|------|------|------|------|
| ekey | 是 | 唯一标识 | `ekey=1` |
| typeid | 是 | 留言栏目ID | `typeid=5` |

---

## 9. 用户相关接口

### 9.1 用户登录

```javascript
import { requestPost } from '@/utils/eyouApi'
import { setUserToken } from '@/utils/auth'

async function login(username, password) {
  const res = await requestPost({
    login: '',
    username,
    password
  })
  if (res.status === 1) {
    setUserToken(res.data.token)
  }
  return res
}
```

### 9.2 获取用户信息

```javascript
async function getUserInfo() {
  const res = await requestPost({
    getUserInfo: ''
  })
  return res.data
}
```

### 9.3 上传图片

```javascript
async function uploadImage(file) {
  const formData = new FormData()
  formData.append('file', file)

  const res = await axios.post(
    '/index.php?m=api&c=index&a=upload',
    formData,
    {
      baseURL: import.meta.env.VITE_API_BASE_URL,
      headers: { 'Content-Type': 'multipart/form-data' }
    }
  )
  return res.data
}
```

---

## 10. 商城相关接口

> 以下接口需 EyouCMS 开启商城功能（45°C商城插件）。

### 10.1 商品列表

使用 `apiList` 接口，指定 `channelid=2`（产品模型）：

```javascript
const res = await callApi('apiList', 'ekey=1&channelid=2&typeid=3&pagesize=12&addfields=price,stock')
const products = res.data.apiList[1].data
```

### 10.2 购物车操作

```javascript
// 加入购物车
await requestPost({
  cartAdd: '',
  goods_id: 10,
  goods_num: 1,
  spec: '' // 规格JSON
})

// 获取购物车列表
await requestPost({ cartList: '' })

// 修改购物车数量
await requestPost({
  cartUpdate: '',
  id: 1,
  goods_num: 2
})

// 删除购物车商品
await requestPost({
  cartDel: '',
  id: 1
})
```

### 10.3 订单相关

```javascript
// 订单列表
await requestPost({ orderList: '', page: 1, pagesize: 10 })

// 订单详情
await requestPost({ orderDetails: '', order_id: '202401010001' })

// 取消订单
await requestPost({ orderCancel: '', order_id: '202401010001' })

// 确认收货
await requestPost({ orderConfirm: '', order_id: '202401010001' })
```

### 10.4 优惠券

```javascript
const res = await callApi('apiCouponList', 'ekey=1&pagesize=10')
const coupons = res.data.apiCouponList[1].data
```

---

## 11. 完整调用示例

### 11.1 项目目录结构建议

```
src/
├── api/
│   ├── index.js        # 接口统一导出
│   ├── article.js      # 文章相关接口
│   ├── product.js      # 产品相关接口
│   └── user.js         # 用户相关接口
├── utils/
│   ├── request.js      # axios 封装
│   ├── eyouApi.js      # EyouCMS 专用方法
│   └── auth.js         # 认证相关
└── views/
    ├── Home.vue
    ├── ArticleList.vue
    └── ArticleDetail.vue
```

### 11.2 按模块封装 API

`src/api/article.js`：

```javascript
import { callApi, requestPost } from '@/utils/eyouApi'

// 获取文章列表
export function getArticleList(params) {
  const { typeid, page = 1, pagesize = 10, keywords = '' } = params
  let paramStr = `ekey=1&page=${page}&pagesize=${pagesize}`
  if (typeid) paramStr += `&typeid=${typeid}`
  if (keywords) paramStr += `&keywords=${encodeURIComponent(keywords)}`
  return callApi('apiList', paramStr)
}

// 获取文章详情（通过单条文档标签）
export function getArticleDetail(aid) {
  return callApi('apiArclist', `ekey=1&aid=${aid}`)
}

// 获取上下篇
export function getPrevNext(aid, typeid) {
  return callApi('apiPrenext', `ekey=1&aid=${aid}&typeid=${typeid}`)
}

// 提交评论
export function submitComment(data) {
  return requestPost({
    commentAdd: '',
    ...data
  })
}
```

### 11.3 Vue 组件中使用

`src/views/ArticleList.vue`（Vue3 Composition API）：

```vue
<template>
  <div class="article-list">
    <div v-for="item in list" :key="item.aid" class="article-item">
      <h3>{{ item.title }}</h3>
      <p>{{ item.description }}</p>
      <span>{{ item.add_time }}</span>
    </div>
    <div class="pagination">
      <button @click="loadPage(current - 1)" :disabled="current <= 1">上一页</button>
      <span>{{ current }} / {{ totalPages }}</span>
      <button @click="loadPage(current + 1)" :disabled="current >= totalPages">下一页</button>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, computed } from 'vue'
import { getArticleList } from '@/api/article'

const props = defineProps({
  typeid: { type: Number, default: 1 }
})

const list = ref([])
const current = ref(1)
const total = ref(0)
const pagesize = 10

const totalPages = computed(() => Math.ceil(total.value / pagesize))

async function loadPage(page) {
  if (page < 1 || page > totalPages.value) return
  current.value = page
  try {
    const res = await getArticleList({
      typeid: props.typeid,
      page,
      pagesize
    })
    list.value = res.data.apiList[1].data
    total.value = res.data.apiList[1].total
  } catch (err) {
    console.error('加载失败:', err)
  }
}

onMounted(() => loadPage(1))
</script>
```

### 11.4 首页批量加载示例

`src/views/Home.vue`：

```vue
<script setup>
import { ref, onMounted } from 'vue'
import { requestApi } from '@/utils/eyouApi'

const siteConfig = ref({})
const navList = ref([])
const banners = ref([])
const newsList = ref([])
const products = ref([])

onMounted(async () => {
  // 一次请求获取首页所有数据
  const res = await requestApi({
    apiGlobal_1: 'ekey=1',
    apiNavigation_2: 'ekey=2',
    apiAdv_3: 'ekey=3&pid=1',
    apiArclist_4: 'ekey=4&typeid=1&row=6&orderby=add_time',
    apiArclist_5: 'ekey=5&typeid=3&row=8&flag=c&addfields=price'
  })

  siteConfig.value = res.data.apiGlobal[1]
  navList.value = res.data.apiNavigation[2].data
  banners.value = res.data.apiAdv[3].data
  newsList.value = res.data.apiArclist[4].data
  products.value = res.data.apiArclist[5].data
})
</script>
```

---

## 12. 常见问题

### Q1: 接口返回 403 / 鉴权失败？

- 检查后台是否开启了小程序 API 接口
- 确认 `apikey` 值是否与后台一致
- 检查 `apikey_token` 生成算法是否正确（参考官方小程序 app.js）
- 确认服务器时间是否准确（token 与时间戳相关）

### Q2: 返回数据为空？

- 检查 `ekey` 是否一致：请求 `apiList_1` 对应 `ekey=1`，返回取 `data.apiList[1]`
- 检查栏目 ID 是否正确，栏目下是否有文档
- 检查 `channelid` 和 `typeid` 是否冲突

### Q3: 跨域问题怎么解决？

EyouCMS 后台默认未开启 CORS，有两种方案：

**方案一：Vite 代理（开发环境）**

```javascript
// vite.config.js
export default {
  server: {
    proxy: {
      '/index.php': {
        target: 'https://your-domain.com',
        changeOrigin: true
      }
    }
  }
}
```

**方案二：后台配置 CORS（生产环境）**

在 EyouCMS 入口文件或 Nginx 配置中添加：

```nginx
add_header Access-Control-Allow-Origin *;
add_header Access-Control-Allow-Methods 'GET, POST, OPTIONS';
add_header Access-Control-Allow-Headers 'Content-Type, Authorization';
```

### Q4: 中文关键词搜索乱码？

确保请求时对关键词进行 `encodeURIComponent` 编码，且 axios 的 `Content-Type` 设置为 `application/x-www-form-urlencoded;charset=UTF-8`。

### Q5: 如何获取文章详情正文？

EyouCMS 标签化 API 中，文章详情可通过 `apiArclist` 指定 `aid` 获取，正文字段为 `content`。如果需要更完整的详情页数据，建议结合 `apiType`（栏目信息）和 `apiPrenext`（上下篇）一起请求。

### Q6: 分页参数怎么传？

`apiList` 接口支持 `page` 参数，直接在参数串中添加 `&page=2` 即可。返回数据中包含 `page`（当前页）、`total`（总条数）字段。

---

## 附录：接口速查表

| 接口名 | 分类 | 用途 | 关键参数 |
|--------|------|------|----------|
| apiGlobal | 全局 | 网站配置 | ekey |
| apiNavigation | 全局 | 导航菜单 | ekey |
| apiChannel | 全局 | 栏目列表 | ekey, channelid, typeid |
| apiType | 全局 | 栏目详情 | ekey, typeid |
| apiAdv | 全局 | 广告列表 | ekey, pid |
| apiAd | 全局 | 单条广告 | ekey, aid |
| apiFlink | 全局 | 友情链接 | ekey, typeid |
| apiList | 列表 | 文档列表(分页) | ekey, typeid, pagesize, page |
| apiArclist | 列表 | 自由文档(无分页) | ekey, typeid, row, aid |
| apiChannellist | 列表 | 栏目分页列表 | ekey, typeid |
| apiCommentlist | 列表 | 评论分页 | ekey, aid, pagesize |
| apiLikearticle | 列表 | 相关文档 | ekey, aid, row |
| apiPrenext | 内容 | 上下篇 | ekey, aid, typeid |
| apiCollect | 内容 | 收藏操作 | aid, opt (POST) |
| apiLike | 内容 | 点赞操作 | aid, opt (POST) |
| apiForm | 内容 | 自由表单字段 | ekey, formid |
| apiGuestbookform | 内容 | 留言表单 | ekey, typeid |
| apiCouponList | 商城 | 优惠券列表 | ekey, pagesize |
| apiSpecnode | 内容 | 专题文档 | ekey, typeid |

---

> **文档说明**：本文件基于 EyouCMS 官方标签化 API 体系整理，部分接口参数可能因版本差异略有不同。
> 开发时请以你当前 EyouCMS 版本的官方文档为准：https://www.eyoucms.com/doc/api/
