## 说明

群组博客，使用的主题是 [HardCandy-Jekyll](https://github.com/xukimseven/HardCandy-Jekyll)。

发布文章请提交到 `_post` 目录下，文件名格式为 `年-月-日-xxx.md`；静态资源请放置到 `assets/post` 目录下，按年份和文章名命名子目录（如果文章只带一张图，可以不新建目录）。

有需求可以提 issue 或直接 QQ 交流。改代码可以直接提交，也可以提 pr。

## 文章字段

| 字段  | 意义 | 是否必须 | 备注 |
| ---- | ---- | ---- | ---- |
| layout | 所属布局 | true | 固定为`post` |
| title | 标题 | true |  |
| subtitle | 副标题 | false | |
| date | 日期 | false | |
| author | 作者 | false | 不设置时将显示为`后端组` |
| authorSite | 作者主页 | false | 文章页作者处超链接的 URL，设置了 `author` 才会生效 |
| color | 文章页面页头颜色 | false | 格式形如`rgb(1,63,148)` |
| tags | 标签 | false | 暂时只支持单个标签 |
| cover | 题图 | false | |
| firstPostShow | 首发超链接的锚文本 | false | 需要与 `firstPostURL` 一同使用 |
| firstPostURL | 首发超链接的 URL | false | 需要与 `firstPostShow` 一同使用 |
