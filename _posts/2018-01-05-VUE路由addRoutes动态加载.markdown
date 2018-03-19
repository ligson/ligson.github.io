---
layout: post
title:  "VUE路由addRoutes动态加载"
categories: 前端
tags: [vue,router,elementui,layout]
author: ligson
description: vue路由动态加载的一种实现方式
---

## 背景

以前做java开发，经常使用的一种权限模型rabc，即把系统的权限分成：用户、角色、权限，不用的用户可以有不同的角色，不用的角色可以有不用的权限，所谓权限在做后台系统中对应的就是系统菜单，通常情况下是用户登录操作，然后查询用户的角色，根据角色在查询出来用户能看到的菜单，但是在使用vue的时候发现必须把vue中对应的路由定义好，所以在使用起来感觉很不方便，难道不能后台请求，然后再动态定义路由吗？

## 思路

vue router2.2.1以上版本提供了addRoutes方法，所以流程是，预先定义一些路由，比如登录的，注册的，然后用户登录，然后请求后台菜单，把请求过来的数据组装成对应的数组，使用addRoutes增加到路由中

##路由 router/index.js定义

```javascript
import Vue from "vue";
import Router from "vue-router";

Vue.use(Router);

export const adminAllRouterList = [
  {
    path: "/admin",
    component: resolve => require(["@/components/Index"], resolve)
  },

  {
    path: "/admin/user",
    component: resolve => require(["@/components/admin/User"], resolve)
  },

  {
    path: "/admin/user/list",
    component: resolve => require(["@/components/admin/user/list"], resolve)
  },

  {
    path: "/admin/user/detail",
    component: resolve => require(["@/components/admin/user/detail"], resolve)
  },
  {
    path: "/admin/role",
    component: resolve => require(["@/components/admin/Role"], resolve)
  }
];

export default new Router({
  routes: [
    {
      path: "/",
      redirect: "/login"
    },
    {
      path: "/login",
      component: resolve => require(["@/components/Login"], resolve)
    },
    {
      path: "/register",
      component: resolve => require(["@/components/Register"], resolve)
    }
  ]
});
```

	注意必须提前把所有的路由对应要跳转到的页面在`adminAllRouterList`中定义一下，require和import的方式我试了，不能直接把页面定义到数据库在加载，有好的办法大家可以分享一下

## 登录页面的定义

```html
<template>
  <div>
    111
    <el-button @click="goToLogin">login</el-button>
    <router-link to="/register">to register</router-link>
  </div>
</template>

<script>
  import ElButton from "../../node_modules/element-ui/packages/button/src/button.vue";
  import { processMenuData } from "@/router/loadRouter";

  export default {
    components: {ElButton},
    name: "HelloWorld",
    data () {
      return {
        msg: "Welcome to Your Vue.js App"
      };
    },
    methods: {
      goToLogin () {
        let routerList = processMenuData();
        this.$router.addRoutes([routerList]);
        this.$router.push("/admin/user");
      }
    }
  };
</script>
```

	用户登录后调用`processMenuData`去加载用户的菜单

## loadRouter文件的定义

```javascript
import menus from "@/assets/menu.json";
import { adminAllRouterList } from "@/router/index";

function __loadData (mData, superPath) {
  let path = superPath ? (superPath + "/" + mData.path) : mData.path;
  if (mData.component) {
    for (let i = 0; i < adminAllRouterList.length; i++) {
      console.log(path);
      if (adminAllRouterList[i].path === path) {
        mData.component = adminAllRouterList[i].component;
        break;
      }
    }
    console.log(path);
    console.log(typeof mData.component);
    if ((typeof mData.component) === "string") {
      console.log(mData.component);
    }
    // mData.component = () => import(mData.component);
  }
  if (mData.children) {
    for (let i = 0; i < mData.children.length; i++) {
      __loadData(mData.children[i], path);
    }
  }
}

export function processMenuData () {
  __loadData(menus, null);
  return menus;
};
```

## menu.json请求数据的定义

```json
{
  "path": "/admin",
  "component": "./components/Index.vue",
  "redirect": "/admin/user",
  "children": [
    {
      "path": "user",
      "component": "./components/admin/user.vue",
      "redirect": "/admin/user/list",
      "children": [
        {
          "path": "list",
          "component": "./components/admin/user/list.vue"
        },
        {
          "path": "detail",
          "component": "./components/admin/user/detail.vue"
        }
      ]
    },
    {
      "path": "role",
      "component": "./components/admin/Role.vue"
    }
  ]
}
```

项目源码:https://github.com/ligson/vrouter,更多技术交流分享请加qq群：450794233
