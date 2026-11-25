# list-macos-installers.sh

从 Apple 官方软件更新目录（sucatalog）中列出**所有可下载的 macOS 完整安装包**，按序号让你选择对应版本，并按需下载 `InstallAssistant.pkg`。

不需要登录 App Store，也不需要 Apple ID；数据全部来自 Apple 公开的苏区目录（swscan），下载直链来自 Apple CDN（swcdn）。

---

## 依赖

| 项目 | 要求 |
| --- | --- |
| 系统 | macOS / Linux（脚本为 POSIX `sh`，实测 macOS 与 Linux 均可） |
| 命令 | `curl`、`awk`、`grep`、`sed`、`sort`（需支持 `-V` 版本排序）、`xargs`（需支持 `-P`） |
| 网络 | 能访问 `swscan.apple.com` 与 `swcdn.apple.com`（约 7 MB 目录 + 每个资产几十 KB 元数据） |
| 磁盘 | 下载阶段需要 12–20 GB 可用空间（各版本安装包体积不同） |

macOS 自带的 BSD 系列工具都满足以上要求，无需安装 GNU 版本。

---

## 安装

```bash
chmod +x list-macos-installers.sh
```

---

## 用法

### 交互式（常用）

```bash
./list-macos-installers.sh
```

执行后会打印带序号的版本列表，再输入序号：

```
==> 可下载的 macOS 版本（共 14 个，新版本在前）：
   1) macOS 27.2.1     build 26B8132
   2) macOS 27.2       build 26B8125
   ...
  14) macOS 15.7.9     build 24G830

请输入要下载的序号 [1-14]: 14
```

选定后输出该版本的 Build 号与 pkg 直链，并询问是否下载：

```
==> 已选择序号 14：macOS 15.7.9 (24G830)
    https://swcdn.apple.com/content/downloads/.../InstallAssistant.pkg

是否现在下载？[Y/n]
```

- 直接回车 = 下载
- 输入 `n` = 只查版本与链接，退出

### 非交互式（适合脚本/批量）

```bash
./list-macos-installers.sh 14        # 直接选第 14 项
printf 'n\n' | ./list-macos-installers.sh 3   # 只看信息，不下载
```

序号来自脚本每次运行时实时生成的列表，**不同时间运行，序号对应的版本会变化**（Apple 会下架旧资产）。需要稳定引用时，请以打印出的版本号 + Build 号为准。

---

## 环境变量

| 变量 | 默认 | 说明 |
| --- | --- | --- |
| `CATALOG` | 内置的 15/14/13/12/10.16… merged catalog 地址 | 换用其他 sucatalog。例如要多列某些新版本线，可换成对应的 `index-26-15-14-13-...merged-1.sucatalog` |
| `JOBS` | `8` | 抓取元数据的并发数。网络差时可降到 `4`，网络好可提到 `16` |

示例：

```bash
JOBS=4 ./list-macos-installers.sh
CATALOG="https://swscan.apple.com/content/catalogs/others/index-26-15-14-13-12-10.16-10.15-10.14-10.13-10.12-10.11-10.10-10.9-mountainlion-lion-snowleopard-leopard.merged-1.sucatalog" ./list-macos-installers.sh
```

---

## 输出与产物

- 列表列依次为：序号、版本号、Build 号；按版本号倒序（新版本在前）。
- 下载文件命名：`InstallAssistant-<版本号>.pkg`（如 `InstallAssistant-15.7.9.pkg`），不会覆盖不同版本。
- 临时文件（目录、元数据 plist）放在 `mktemp` 目录，脚本退出时通过 `trap` 自动清理。

### 下载之后

1. 双击 pkg 安装（或 `sudo installer -pkg InstallAssistant-15.7.9.pkg -target /`），安装完成会在 `/Applications` 生成 `Install macOS <名称>.app`。
2. 制作 U 盘启动盘：

```bash
sudo /Applications/Install\ macOS\ <名称>.app/Contents/Resources/createinstallmedia \
  --volume /Volumes/<U盘卷名>
```

需要**降级安装**时，只能在对应机型支持的系统范围内操作，且通常要抹盘；跨大版本降级请提前备份。

---

## 工作原理

1. 拉取 sucatalog（一个 XML plist），从中筛出所有含 `com_apple_MobileAsset_MacSoftwareUpdate.plist` 的资产地址（排除 `integrityDataV1` 这类签名文件）。
2. 对每个资产地址并发抓取其元数据 plist，用 `plist_get()` 取 `OSVersion` 与 `Build`。
3. 把资产地址中的 `com_apple_MobileAsset_MacSoftwareUpdate.plist` 替换为 `InstallAssistant.pkg`，得到可直接下载的安装包直链。
4. 汇总为 `版本 / Build / 直链` 三列，按版本倒序排序；同一版本存在多个 Build 时只保留 Build 号最大的一条。
5. 按你输入的序号取出对应行，打印版本与链接，确认后用 `curl -L` 下载到当前目录。

---

## 已知限制

- **只列完整安装包（InstallAssistant）**，不包含增量更新、安全响应更新（RSR）、Beta 资产。
- 某个资产若元数据里没有 `OSVersion`（少数早期或特殊机型资产），会被跳过，因此总数可能略少于目录中的资产数。
- 列表的“新版本在前”依赖 `sort -V` 对版本号做自然排序；版本号格式异常（带后缀、非数字段）时排序结果可能不理想，但不会漏项。
- Apple 随时会下架旧资产，历史版本不一定还能下载；此时换更全的 `CATALOG` 或改用 Archive 渠道。
- 脚本不做校验和/签名校验，下载后请自行确认 pkg 由 Apple 签发（`pkgutil --check-signature`）。

---

## 常见问题

**Q：提示 “目录中没有找到 InstallAssistant 资产”**
A：多为网络被拦或返回了错误页。先单独验证目录可访问：

```bash
curl -fsSL --max-time 20 -o /tmp/cat.plist "$CATALOG" && grep -c 'MobileAsset_MacSoftwareUpdate' /tmp/cat.plist
```

返回 0 说明 `CATALOG` 地址过期，换一个 merged catalog 地址。

**Q：列表里没有我要的版本（比如某个 12.x 老版本）**
A：merged catalog 只保留 Apple 尚在挂的资产。老版本需要用更完整的目录地址，或 Apple 官方的 archived 安装包渠道。

**Q：下载很慢或中途断开**
A：`curl` 已带 `--retry 3`；仍失败可把打印出的直链复制出来，用下载工具续传。

**Q：`sort: illegal option -- V`**
A：你的 `sort` 不是 BSD/GNU 现代版本（常见于 BusyBox）。安装 `coreutils` 或改用 macOS 自带环境执行。

> （注：部分内容由豆包 AI 生成）