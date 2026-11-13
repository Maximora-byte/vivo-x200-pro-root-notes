# 发布收据

- 公开仓库：https://github.com/Maximora-byte/vivo-x200-pro-root-notes
- 博客正文：https://blog.maximoraverse.org/posts/vivo-x200-pro-temporary-root-field-notes/
- 正文源版本：`48fff871bf8837d77337914902795fe4255dffa0`。
- 博客 Markdown SHA256：`21622e2a30318acfea842bcb4e786ec9dfb8f33f00516114939cf5ab09cd4b24`。

公开前做了去标识、JSON、仓库内部链接和Git空白检查。博客按当前生产源码的隔离副本构建，Astro检查与构建通过；仅更新新文章及必要的列表、RSS、站点地图和Pagefind发现文件，没有用新构建覆盖全部历史页面。

发布变更13个文件（6个替换、7个新增），566个非目标文件与39个非目标HTML字节一致。路由配置与容器保持不变，私有端留有回滚字节和前后摘要；访问资料不进入本仓库。

公开文章、首页、归档、RSS、站点地图与搜索入口HTTP200；浏览器Pagefind搜索watchdog能找到新文。桌面1280px、手机390px和320px检查未见页面横向溢出，正文十章可读。实际页面验证与构建通过不等于GitHub托管CI通过，本仓库没有作该项声明。

检查也发现一项既有问题：站点公共脚本会动态加载一个缺失的命令面板资源而产生404，同一问题在未改动的旧文章上复现。此次没有扩大范围修改公共功能，正文与文章搜索可用。结构化收据见 [publication-receipt.json](../evidence/publication-receipt.json)。
