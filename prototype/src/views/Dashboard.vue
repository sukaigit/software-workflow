<template>
  <div class="page-body">
    <div style="display:grid;grid-template-columns:repeat(3,1fr);gap:16px;margin-bottom:24px">
      <div class="card"><div class="card-title">今日新增</div><div style="font-size:32px;font-weight:700;color:var(--color-primary)">0</div></div>
      <div class="card"><div class="card-title">待处理</div><div style="font-size:32px;font-weight:700;color:var(--color-warning)">0</div></div>
      <div class="card"><div class="card-title">累计总量</div><div style="font-size:32px;font-weight:700;color:var(--color-text)">0</div></div>
    </div>
    <div class="card">
      <div class="card-title">欢迎使用管理系统</div>
      <p style="color:var(--color-text-secondary);font-size:14px;line-height:1.8">
        本模板基于 Vue 3 + Vite，Apple 风格设计。<br>
        请在 <code style="background:var(--color-bg);padding:2px 6px;border-radius:4px;font-size:13px">src/router/index.js</code> 添加你的业务页面路由，<br>
        参考 <code style="background:var(--color-bg);padding:2px 6px;border-radius:4px;font-size:13px">src/views/ExamplePage.vue</code> 创建 CRUD 页面。
      </p>
    </div>
  </div>
</template>
<script>
export default {}
</script>