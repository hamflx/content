# Chrome 时代的浏览器兼容性测试

对于这个标题，我想有一些人是很有疑惑的，对于 `IE` 时代来说，确实需要去研究浏览器的兼容性，但是对于浏览器中的老大 `Chrome`，我们需要考虑它的兼容性吗？

就我目前做的项目，目标用户都是使用 `Chrome` 浏览器，但是仍然在工作中碰到很多关于 `Chrome` 的问题，比如：

- 用户反馈你们的软件开了一天就卡的不行，有时候半天不到就会崩溃，然后你吭哧吭哧去解决内存泄漏的点，然后自己测试发现没问题，但是一上线，客户又反馈卡卡卡（浏览器内存泄漏）；
- 用户机器上，表格的渲染有时缺一某一条边框，有时候有的边框特别粗；
- 明明没有设置 `z-index`，或者设置了较小的 `z-index`，但是滚动到容器下面的时候，被遮住的那部分滚动条却突破了容器显示出来了；
- 开发时没考虑 `API` 兼容性，用了 `replaceAll`，然后在用户机器上挂掉了；
- 因为要嵌入第三方页面，考虑到 `Chrome` 变更了 `SameSite` 策略，默认为 `None`，需要兼容新旧版本的 `Chrome`；
- `Chrome` 删除了 `/deep/` 的支持，导致项目中错误使用 `/deep/` 的地方样式崩溃；
- `Chrome` 在某个版本删除了 `Event.path` 的支持，然后用这个的地方也都崩了。

上述情况有些是属于 `bug`，有些是浏览器更新太快而开发人员知识没有跟上，还有些就是单纯的开发人员的失误。

像 `Event.path` 实际上是一个非标准属性，在 `Chrome 100` 上还可以正常使用，在 `Chrome 101` 的时候，如果你使用这个属性，会在 `Issues` 里输出一个 `Deprecated Feature Used` 的警告。然后在 `Chrome 104` 里就不能用了。然而你高强度开发的时候，真的会在意这个警告吗？

所以，浏览器的兼容性问题一直都在，只不过，从兼容 `IE` 变换为兼容老版本 `Chrome` 和兼容新版本 `Chrome`，以及兼容特定版本 `Chrome`。

因此，你可能会需要频繁的下载不同版本的 `Chrome` 来做各种兼容性测试，这之前对我来说是个比较麻烦的事，我不喜欢通过第三方网站下载的安装包，通过官方 `Chromium` 给出的方法又太麻烦。

后来无意间看到这个仓库：[google/fetchchromium](https://github.com/google/fetchchromium)，但是这个工具需要提供 `revision`，不能通过版本号下载，然后我就自己搞了一个 [hamflx/fetchbrowser](https://github.com/hamflx/fetchbrowser)，可以通过给定版本号下载特定的 `Chromium` 浏览器，另外，也支持下载 `Firefox`。

## fetchbrowser

仓库地址：<https://github.com/hamflx/fetchbrowser>，欢迎 `star` (❁´◡`❁)。

安装方式：

```powershell
irm https://raw.githubusercontent.com/hamflx/fetchbrowser/master/install.ps1 | iex
```

下载 `Chromium 98`：

```powershell
fb 98
```

**注意：在特定平台第一次下载 `Chromium` 会比较慢，因为会联机查找版本信息，后续会使用缓存的数据。**

下载 `Chromium 109.0.5414.120`：

```powershell
fb 109.0.5414.120
```

下载 `Firefox 98`：

```powershell
fb --firefox 98
```

对 `Firefox` 的支持可能会有问题，因为 `Firefox` 官方只提供了安装包的安装形式，这里是下载了官方的安装包后解压实现的。
