# lx-music-mobile v1.9.2 适配 CarWith / Jovi InCar —— 功能实现与重置说明

改动基线：上游 `lyswhut/lx-music-mobile` tag **v1.9.2** = 提交 `0669d1b6d419967822ff52ec06e50b9613bfd136`
配套源码：`lx-music-mobile-1.9.2-适配CarWith-纯净源码.zip` —— 解压得到
`lx-music-mobile/`（上游 v1.9.2 + 22 处 `src/` 改动，目录内没有任何额外文件）、
`lx-music-1.9.2-CarWith-src-原包名-cn.toside.music.mobile.patch`（本源码树对应的那片补丁，正/反向都能用）与 `SHA256SUMS.txt`（759 条）
源码树共 758 个文件 = 官方 751 + 新增 7；已实测：这片补丁正向打到官方 v1.9.2 树上，结果与交付的源码树**逐字节一致**，反向撤销后再 `diff -rq` 官方 = 0 行差异
包名口径：**只用上游原包名 `cn.toside.music.mobile`**，本文不涉及也不推荐任何改 applicationId 的变体。
         少数「成品实测值」（APK 大小、条目数、个别 sha256）取自遗留的两份旧成品，同代码同签名、只有包名不同；
         本文只在 §A1 说明一次，后面统称「遗留成品」，不再展开。

---

## 这份文档怎么读

**第一部分（§A*）讲怎么实现的** —— 从「用户能感知到什么」一路走到 smali 的第几行改成了什么。
**第二部分（§B*）讲怎么重置** —— 正向重做、逐层撤销、每步用什么命令验证、上游出新版怎么迁移、故障速查。
**附录**是 native 层与 dex 注入的逐字源码，脱离工具包也能照着重建。

三个词在本文里的固定含义：

| 词 | 指什么 |
| --- | --- |
| **源码树** | 随本文交付的那份纯源码：`v1.9.2` + 22 处 `src/` 改动。可以直接放进 git 仓库，里面没有施工脚本、没有产物、没有说明文件 |
| **工具包** | 先前那份《源码与说明》交付包：`scripts/`（施工脚本 + `工具与依赖.txt` + `android-debug.keystore`）、`native/`（Java 源码 + 注入前后 smali + 三份 diff）、`patches/`（两份遗留 git 补丁，本文口径用不到）、`prebuilt/`（两份 bundle 与 `classes2.dex`）、`docs/`（《复核报告-改动面审计》《原说明书》《APK哈希参考》《修改说明》）。本文按这些名字引用它们；本文附录已内嵌其中不可再生的部分 |
| **基线树** | 官方 `v1.9.2` 的源码树，本文所有「和官方比」的检查都以它为尺子（取法见 §B2.3） |

包名口径：源码树里 `src/config/constant.ts:98` 保持上游原值 `cn.toside.music.mobile.provider`，
**换包名是构建期由脚本注入的**，不是仓库状态。所以这份源码可以直接同步进仓库，不用为包名开分支。

## 目录

- §A1 这段改动是什么 · §A2 四层改动地图与数据流 · §A3 JS/TS 层逐文件 · §A4 bundle 层
  · §A5 dex 层（classes.dex 注入 + classes2.dex） · §A6 打包层（AXML/arsc/签名）
  · §A7 从零重做的依赖与步骤 · §A8 脚本每一步在断言什么 · §A9 已知坑与口径纠正
  · §A10 术语表 · §A11 关键文件与校验值
- §B0 一分钟版 · §B1 四层的重置手段 · §B2 前置检查与完整性自检 · §B3 从源码重做出 APK
  · §B4 逐层重置（撤销） · §B5 验证矩阵 · §B6 升级 checklist · §B7 故障速查
  · §B8 校验值与可抄文字 · §B9 交接三句话
- 附录 A `CarLinkBridge.java` 全文 · 附录 B `ContentObserver.java`（编译期桩类）
  · 附录 C 三处 dex 注入的逐字 diff · 附录 D 新增 7 个源码文件的作用与行数 · 附录 E 设置项与车机协议表

---
# 第一部分 功能实现说明

## §A1 这段改动是什么

上游洛雪音乐 v1.9.2 不认识 CarWith / Jovi InCar（vivo 车载互联）。它往安卓 MediaSession 元数据里塞的
只有 `android.media.metadata.LYRICS` 这类标准键，而 CarWith 那侧要读的是 `ucar.media.metadata.*` 专用键；
主题那边也没有任何「跟着车机走」的入口。本补丁补上两类能力，共 3 个新设置项：

| 设置项 key | 出现位置 | 文案 |
| --- | --- | --- |
| `player.isShowCarWithLyric` | 设置 → 播放 | 显示CarWith / Jovi InCar歌词 |
| `common.followCarWithTheme` | 设置 → 主题（在「跟随系统亮、暗模式切换主题」下面） | 跟随CarWith亮、暗模式切换主题 |
| `common.followSunTheme` | 同上，再下面一行（带一行状态小字） | CarWith连接时跟随日出日落切换主题 |

深浅色的判定是三个来源的「或」：手机系统深色、车机自报深浅色、按所在城市日出日落估算。
三个开关互相独立，任意组合都能用。

包名口径只有一个：上游原包名 `cn.toside.music.mobile`。源码树不改一行、构建也不换 applicationId，
本文不讲任何变体包名，后文出现的实测值都按上面这条口径理解。

| 项 | 值 |
| --- | --- |
| applicationId | `cn.toside.music.mobile`（= 上游，不改） |
| provider authority | `cn.toside.music.mobile.provider`（`src/config/constant.ts:98` 保持上游原值） |
| versionCode / versionName | 79003 / 1.9.2（来历见 §A7.4） |

> 文中少数「成品实测值」（APK 大小 14,265,630 B、条目数 799、个别 sha256）取自遗留的两份旧成品：
> 同代码、同签名、同 versionCode，只有 applicationId 不同。它们只是改动面与量级的参照，
> **不是**你要验收的文件 —— 原包名的 APK 至今没构建过（见 §B3.1 末）。要出它：照 §A7 / §B3 走，
> 并按 §B3.1 先把两处对撞的断言条件化。后文一律称这两份为「遗留成品」，不再另行解释。

签名是安卓公签（= 上游 `android/app/debug.keystore` 逐字节同一把），证书 SHA-256
`fac61745dc0903786fb9ede62a962b399f7348f0bb6f899b8332667591033b9c`，v1 + v2 方案。
用公签是刻意的：同签名才能覆盖安装，换钥匙装上去就是「另一个应用」，用户歌单/音源全丢。
应用名沿用上游 `LX Music`，没有改。minSdk 21 / targetSdk 29。

---

## §A2 一分钟看懂的改动地图

改动落在**四个层次**上，越往下越脏（越难被上游合并）。前两层是可读源码，后两层是逆向产物。

```
┌ A. JS/TS 源码层（23 文件，+789 / −9）──────────────── 本源码树 + patches/*.patch
│   3 个新设置项、7 个新文件、3 份语言包各 +10 条文案、主题判定改「或」
└──────────────┬─ Metro + hermesc ──────────────┐
               ▼                                │
┌ B. JS bundle 层（assets/index.android.bundle）│ prebuilt/index-<包名>.android.bundle
│   provider authority 烤在字节码里 ⇒ 每个包名一份
└──────────────┬────────────────────────────────┴──┐
               ▼                                   │
┌ C. dex 层 ────────────────────────────────────────┴─ native/smali/, native/java/, prebuilt/classes2.dex
│   classes.dex：注入 3 个文件（l4/d、k4/b、MainApplication）
│   classes2.dex：新增，恰好 3 个类（CarLinkBridge 及其内部类）
└──────────────┬─
               ▼
┌ D. 打包层 ─────────────────────────────────────────
│   AndroidManifest.xml / resources.arsc 二进制换包名 → 重打包 → zipalign → v1+v2 签名
└────────────────────────────────────────────────────
```

数据流（一次「当前句歌词上屏」和一次「车机切深色」）：

```
JS 设置项 player.isShowCarWithLyric
   └─ src/core/init/player/lyric.ts / src/core/lyric.ts 走「车机分支」
        → updateNowPlayingTitles({title, artist, album, lyricLine})
             → NativeTrackPlayer / Bundle 键 "lyricLine"
                  → k4/b.smali (MetadataManager.v(Bundle))  读 bundle → 写 ucar.media.metadata.LYRICS_LINE / LYRICS_WHOLE
                  → l4/d.smali (TrackMetadata)              读 "lyricLine" 存字段 j → 建 MediaMetadataCompat 时写两个 ucar 键
                       → MediaSession 元数据 → CarWith 车机端按专用字段逐句显示

Settings.Global "ucar_night_mode"（车机写，0=浅色 1=深色）
   └─ classes2.dex: CarLinkBridge$NightObserver（原生 ContentObserver）
        → 落到文件 carlink_state.txt，格式 `连着(0/1)|深浅(-1|0|1)`
             → JS: src/utils/sunState.ts 每 2 秒读一次
                  → src/utils/sunLocation.ts（IP 定位）+ src/utils/sunrise.ts（日出日落算法）
                       → src/theme/themes/index.ts 里 dark = 系统深色 || (sunState.enabled && sunState.night)
                            → 主题切换
```

---

## §A3 A 层：JS/TS 源码改动逐文件说明

**共 23 个文件：7 个新增、16 个修改**（+789 / −9）。全部体现在 `patches/lx-music-1.9.2-CarWith-src-<包名>.patch`。
注意两份补丁只差一行（见下 `src/config/constant.ts`）。
改动的原始驱动是 `scripts/patch191_lyric_js.py`（歌词线）与 `scripts/patch191_suntheme_js.py`（主题线），
它们用「独一无二锚点 + 出现次数必须为 1」的方式改源码，锚点不唯一就直接报错退出 —— 这也是为什么
1.9.1 → 1.9.2 能平移：上游那 6 个提交没碰这些锚点。

### §A3.1 新增的 7 个文件

| 落到工程的路径 | 行数 | 正文来源 | 作用 |
| --- | --- | --- | --- |
| `src/utils/sunState.ts` | 10 | `scripts/js4/sunState.ts` | 模块级可变状态 `{enabled, night}`，给主题计算和车机轮询之间传值；刻意不用 redux，避免每次轮询触发全局重渲染 |
| `src/utils/sunrise.ts` | 90 | `scripts/js4/sunrise.ts` | 内置日出日落算法，按经纬度+日期现算升落时刻；极圈算不出时上层显示「极圈地区」 |
| `src/utils/sunLocation.ts` | 294 | `scripts/js4/sunLocation.ts` | IP 定位 + 调度策略 + 状态文案；缓存 key `@salt_sun_location`、`@salt_sun_budget`（`:26-27`） |
| `src/core/init/sunTheme.ts` | 120 | `scripts/js4/sunTheme.ts` | 由 `init/index.ts` 注册的那条线：每 2 秒读 `carlink_state.txt`（`:13-31`）、维护 sunState、驱动状态行 |
| `.../Theme/FollowCarWithTheme.tsx` | 34 | `scripts/js4/FollowCarWithTheme.tsx` | 主题页「跟随CarWith亮、暗模式切换主题」那一行开关 |
| `.../Theme/FollowSunTheme.tsx` | 101 | `scripts/js4/FollowSunTheme.tsx` | 「CarWith连接时跟随日出日落切换主题」开关 + 下面那行五态状态文字 |
| `.../Player/IsShowCarWithLyric.tsx` | 48 | 由 `patch191_lyric_js.py` 生成（不在 js4/） | 播放设置那一行车机歌词开关；勾上时 `showRemoteLyric(true)`，失败就把开关落回 false |

`js4/` 里那 6 个文件与 1.9.1 那条线**逐字节相同**（说明书附录 A 已核对），换版本时不用重看正文。

`src/utils/sunLocation.ts` 的定位策略（这些约束是有意的，改动前请一并理解）：
**只在连着 CarWith 期间发请求**；刚连上查一轮；平时 2 小时一轮；进入日出日落前后 90 分钟内加密到 30 分钟一轮；
三个 IP 定位源并发请求；全部失败后 15 分钟内不再试；一分钟内最多一轮；**只用 HTTPS**。
断开连接时一个请求都不发。位置不缓存时刻表，也不让人手填。

状态行句式固定为 `IP 城市 · 时间 日出/日落 · 状态`，状态五态：定位成功 / 上次定位 / 定位失败 / 默认定位，
外加极圈情况下把时间格换成「极圈地区」。

### §A3.2 修改的 16 个文件

**设置项、类型与包名相关**
- `src/config/defaultSetting.ts`（+6）：锚在 `'common.isAutoTheme': false,`（该串在 1.9.2 全树只出现 1 次）之后，
  插入 `common.followCarWithTheme` / `common.followSunTheme` 两个默认 false，各带一行注释。
- `src/types/app_setting.d.ts`（+7）：锚在 `'common.isAutoTheme': boolean` 之后补两个布尔声明；
  播放侧另在 `player.isShowBluetoothFullLyric` 附近补 `player.isShowCarWithLyric`。
- `src/config/constant.ts`（改 1 行，`:98`）：**唯一与包名绑定的源码改动**，
  `APP_PROVIDER_NAME = 'cn.toside.music.mobile.provider'` → `'<PKG>.provider'`
  （`patch191_lyric_js.py:131-133`）。它必须与 `rebuild191.py:115-116` 的 `NEW_PACKAGE` 同串，
  因为它编进 Hermes 字节码 —— 所以 bundle 必须与包名配套。

**主题计算与初始化**
- `src/theme/themes/index.ts`（+10/−4）：原来只在 `common.isAutoTheme && shouldUseDarkColors` 时取 `'black'`；
  改成 `const dark = (isAutoTheme && shouldUseDarkColors) || (sunState.enabled && sunState.night)`，深色时用 `theme.darkId`；
  并在 `import themeState from '@/store/theme/state'` 后补 `import { sunState } from '@/utils/sunState'`。
- `src/core/init/index.ts`（+2）：`import initSunTheme from './sunTheme'` 并注册进初始化序列。
- `src/screens/.../Theme/index.tsx`（+4，`:6-7`、`:18-19`）：`<FollowCarWithTheme />`、`<FollowSunTheme />`
  紧跟在 `<IsAutoTheme />` 之后 —— 这就是「在跟随系统那行下面」。

**歌词/元数据通路**
- `src/plugins/player/utils.ts`（+2，`:175-177`）：`NowPlayingTitles` 类型在 `lyric?: string` 后加
  `lyricLine?: string`（注释：当前这一句，交给车机端逐句显示）。
- `src/plugins/player/playList.ts`（+6/−2）：`:36` `getCurrentFullLyric` 的门控扩成
  `isShowBluetoothFullLyric || isShowCarWithLyric`；`:218-219` 车机模式下标题还给歌名（原版会把 `lastLyric` 顶成标题）；
  `:233` 在 isCarWith 时附 `...(isCarWith ? { lyricLine: playerState.lastLyric ?? '' } : {})`。
- `src/core/init/player/lyric.ts`（+16）：`:7,11-19` 逐句回调里加车机分支并 `return` 短路，
  走 `updateNowPlayingTitles({title, artist, album, lyricLine: lrc ?? ''})`，不动 `lrc == null` 的原逻辑；
  `:54-57` 开机时按开关拉起逐句事件源。
- `src/core/lyric.ts`（+6，`:43-47`）：切歌时送一次 `{lyric, lyricLine: ''}` —— 整首单独送，车机可提前排上下句。
- `src/screens/.../Player/index.tsx`（+2，`:11,30`）：`<IsShowCarWithLyric />` 挂在 `IsShowBluetoothLyric` 之后、
  `IsShowBluetoothFullLyric` 之前。
- `src/screens/.../Player/IsShowBluetoothLyric.tsx`（+3/−2，`:14,25,26`）：关键联动修复。逐句事件源是两行共用的，
  改成 `showRemoteLyric(isShowBluetoothLyric || settingState.setting['player.isShowCarWithLyric'])`，
  清理条件同样取或（两个都关才停并回写 title），否则关蓝牙会把车机歌词一起停掉。
- `src/plugins/sync/client/client.ts`（−2，`:136-138`）：删掉 `import { toast } …` 和 `toast('Sync connected')`。
  与本功能无关，是那条线的历史顺手项；只要最小改动的话可以还原它（还原后需重编 bundle）。

**语言包（3 份 × 10 条 = 30 个 key，无改动无删除）**
- `src/lang/zh-cn.json` / `zh-tw.json` / `en-us.json`：主题块 9 条插在 `"setting_basic_theme_auto_theme"` 之后，
  歌词条 1 条插在 `"setting_play_show_bluetooth_lyric"` 之后（三份同位，成品行号 299-307 与 372）。
  key：`setting_play_show_carwith_lyric`、`setting_basic_theme_follow_carwith`、`setting_basic_theme_follow_sun`、
  `setting_basic_theme_sun_rise`、`setting_basic_theme_sun_set`、`setting_basic_theme_sun_loc_info` / `_polar` / `_stale` / `_failed` / `_default`。
  `patch191_lyric_js.py` 的 `lang()` 是幂等的（`:36-39`），主题侧的重复插入会直接报错（`patch191_suntheme_js.py:140-141`）。
- 两个不在流水线里、但改文案前后值得手工跑的闸门：
  `scripts/measure_text_width.py` 按语言包真实文案估排版宽度（状态行 12sp / 可用 310dp，开关名 14sp / 可用 296dp，
  系数：CJK 1.0em、数字 .56、大写 .66、空格 .28、其余 ASCII .52），量 3 个开关名 + 5 条状态行 —— 是「会不会折行」的预估，不是像素级结论；
  `scripts/check_lang_params.py` 把 `FollowSunTheme.tsx` 里 `t('key', {…})` 的实参与语言包模板 `{占位符}` 对撞，
  不一致 `exit 1`，另含「info 行不得留 `{source}`」这条规则。**它必须带工程树作参数**，否则会退回 `BASE/src191` 快照。

---

## §A4 B 层：bundle

`scripts/bundle191.py` 里 `PKGS` 那个元组是脚本自带的默认值（`:72-78`；`--pkg` 只编一份、
`--keep` 不清树），走原包名时只给一个值 `PKG=cn.toside.music.mobile`。它的流程：设 `PKG` → 重跑两个 JS 打补丁驱动
（于是 `constant.ts` 换成对应 authority）→ Metro 打 JS → 工程自带 hermesc 编字节码。
产物命名 `js191/index-<包名>.android.bundle`（`:156-157`），元组第一份还会额外复制成通用的
`index.android.bundle`（`:158-159`）。

⚠ `prebuilt/` 里那两份现成字节码**都不是原包名这份**。provider authority 烤在 Hermes 字节码里，
所以原包名的 bundle 只能现编（§B3.2 的 B1 两步，或 `bundle191.py`），抄 `prebuilt/` 会被
`rebuild191.py:67-72` 当场拦下。

确切命令（`bundle191.py:135-137` 与 `:149-150`）：

```bash
# ① Metro（cwd = 工程树）
node <tree>/node_modules/@react-native-community/cli/build/bin.js bundle \
     --platform android --dev false --entry-file index.js \
     --bundle-output <out>/index.android.bundle
# ② hermesc（cwd = 输出目录，只能用相对文件名）
hermesc -emit-binary -max-diagnostic-width=80 -out index.hbc index.android.bundle
```

必须知道的三点：

1. **hermesc 会把输入文件名原样记进字节码**。必须 `cd` 到输出目录、用相对文件名调用；
   用绝对路径调用会把你本机目录烤进成品。第 [8] 步就是查这个（`rebuild191.py:519-523`，禁串 `:\Users\`、
   `\\Users\\`、`补丁重打工具`）。
2. **平台差异**：react-native 0.73.11（实测工程 `package.json:57`）自带 `sdks/hermesc/{win64-bin,linux64-bin,osx-bin}` 三份，
   脚本按 `sys.platform` 挑（默认落 linux64），可用 `HERMESC` 环境变量覆盖；文件不存在报错，不可执行会自动 `chmod 755`。
   原件说明书写死 win64，这条已改成跨平台。
3. **`node_modules` 必须是非空真目录**：`_real_deps()`（`:37-52`）会查 Windows 的
   `FILE_ATTRIBUTE_REPARSE_POINT(0x400)`，用 junction/联接点指向别处的 `node_modules` 判为不可用 ——
   Metro 解析不到 `buffer` 这类 polyfill。

---

## §A5 C 层：dex 改动

### §A5.1 classes.dex —— 恰好 3 个文件

挂钩表在 `rebuild191.py` 的 `EDITS_BY_FILE`，**8 处锚点 = 4（l4/d）+ 3（k4/b）+ 1（MainApplication）**，
每条必须**恰好命中 1 次**，否则第 [3] 步直接退出（`:288`）。

**`l4/d.smali`（TrackMetadata，4 处）** —— 它负责把元数据组装成 `MediaMetadataCompat`。
1. 新增字段 `.field public j:Ljava/lang/String;`（存当前句）。官方该类实例字段占满 `a`–`i`
   （实测 `/tmp/smali_192/l4/d.smali:7-23`），所以 `j` 不撞名。
2. 在 `:cond_76` 前插 4 行：从 bundle 读 `"lyricLine"` → `BaseBundle->getString` → `move-result-object` → `iput-object` 到 `j`。没传就是 null，不覆盖。
3. 原来「整首歌词为空就直接跳到 `:cond_31`」改成 `if-eqz v1, :cond_ucar`，让当前句仍有机会写。
4. `:cond_ucar` 段：只有 JS 真送了 `lyricLine`（= 勾了车机歌词）才写
   `ucar.media.metadata.LYRICS_LINE`（取 `j`）与 `ucar.media.metadata.LYRICS_WHOLE`（取 `h`），
   经由 `MediaMetadataCompat$b;->e(String,String)`。
   这样**开关关着时行为与原版逐字一致**，也不会被「蓝牙完整歌词」顺带触发。
   （写这两个键必须排在 `setMetadata` 之前，第 [8] 步会检查这个顺序。）

**`k4/b.smali`（MetadataManager，3 处）** —— 每句歌词都经过 `updateNowPlayingTitles → service/a.h → k4/b.v(Bundle)`，
它自己维护一个构建器（字段名 `k`，实测 `smali_192/k4/b.smali:27`），原版只认 title/artist/album/lyric，所以这里也要补。
1. `# static fields` 加 `.field static ucZ:Z`：记「上一次发布有没有送过 lyricLine」。
2. `.method public v(Landroid/os/Bundle;)V` 的 `.registers 5` → `6`（新逻辑多要用一个寄存器）。
3. 在 `:cond_5` 之后插 32 行：读 `"lyricLine"`；非空则读 `"lyric"` 并按需写 `LYRICS_WHOLE`、必写 `LYRICS_LINE`，
   `sput-boolean ucZ=1`；为空则跳到 `:cond_ucar_off`，**只有 `ucZ` 为真时**才把两个键清成 `""` 并把 `ucZ` 落回 0。
   即「关着 == 原版，一行都不多写」，且只在从送过翻到没送时清一次，不会每句都写。
   注释里还专门记了寄存器类型一致性：`v0` 空串 / `v1` int / `v2` 构建器 / `v3` 键名，两条路在 `:cond_ucar_done`
   汇合时校验器才不会报类型冲突。

**`MainApplication.smali`（1 处）** —— `invoke-super {p0}, Ls4/d;->onCreate()V` 之后紧跟
`invoke-static {p0}, Lcom/salt/music/bridge/CarLinkBridge;->start(Ljava/lang/Object;)V`。只用 `p0`，不加寄存器。
`--probe` 模式下会再多一行 `DayNightProbe;->start`（见§A8 节）。

混淆类名的认法：`l4/d` = TrackMetadata、`k4/b` = MetadataManager、`MainApplication` 的父类仍是 `s4/d`。
1.9.1 → 1.9.2 这些一个都没变，`:cond_76` / `:cond_31` / `:cond_5` 标签编号也没变。
另外 baksmali 在「目录名与某个类文件同名」时会给目录加 `.1` 后缀（1.9.1 那支是 `l4.1/d.smali`、`k4.1/b.smali`；
1.9.2 无冲突，直接 `l4/`、`k4/`），脚本两种都试，找不到就报「多半是混淆类名/目录名变了」。

### §A5.2 classes2.dex —— 新增，恰好 3 个类

源码 `native/java/CarLinkBridge.java`（282 行，`sha256` 前缀 `ed3578886e653bb1`）：
`public final class CarLinkBridge` + 匿名 `Runnable`（`:55` → `$1`）+ `private static final class NightObserver`（`:65` → `$NightObserver`）。
用 ecj 3.21.0（`-source/-target 1.8 -proc:none -nowarn`）编 class，再用 r8 1.6.84 的 D8（`--min-api 21`）出 dex。
`native/java/ContentObserver.java`（16 行）**只是编译期桩类**：让桥接不依赖 android.jar 也能编过，
第 [4b] 步过滤 `android/` 前缀并在事后断言它绝不进 dex（`classes2.dex 里混进了框架/依赖类`）。
桩类的方法签名字面量必须与真类逐字符一致，否则运行时 `NoSuchMethodError`。

运行时序（`CarLinkBridge.java` 行号）：

- `start(ctx)` `:49-60`：`started` 幂等；起一条 `name="CarLinkBridge"` 的**守护**线程跑 `loop(ctx)`。
- `loop()` `:86-128`：`while(true)`，整轮 `try { … } catch (Throwable ignored)` ——
  **任何一次失败都不退出循环**（`:80-85` 注释：线程一死，状态文件就永久停在旧值，两个跟随功能静默失效）。
  顺序：反射 `getExternalFilesDir(null)` 取目录 → `getSystemService("display")` →
  目录先到就先写一次 `0|-1` 盖掉跨进程残留（`:96-105`，JS 是 2 秒一轮，会抢在本线程第一轮之前读到旧值）→
  file 与 display manager 都齐了且未挂观察者时 `observe(ctx)` → `isCarWithUp(dm)` 为真才读 `nightMode(ctx)`，
  否则写 `0|-1` → **与新值不同才 `write()`**（一直连着且不切深浅色时一次写都不做）→ `sleep(TICK_MS)`，`TICK_MS = 5000L`（`:35`）。
- `isCarWithUp()` `:177-199`：`getDisplays()` 遍历，**必须 `getType()==5`（DISPLAY_TYPE_VIRTUAL）
  且 `getName().toLowerCase()` 含 `carlink` 或 `ucar`** —— 只看名字会误判。
- `nightMode()` `:161-176`：反射 `Settings$Global.getInt(resolver, "ucar_night_mode", -1)`。
  语义：**0=浅色、1=深色、-1=读不到**（未装 CarWith / 键不存在 / 查询失败）；`-1` 时 JS 退回日出日落那条。
  键名取自 com.miui.carlink 4.0.20 里的字面量。
- `NightObserver` `:65-78`：`super(true)`（threadAware，回调在主线程派发），**只**置 `sWake=true` + `WAKE.notifyAll()`
  叫醒轮询线程，绝不碰磁盘；`sObserver` 留强引用（`:44`），被 GC 系统就不再通知。
  匿名 `Runnable`（`$1`）的职责只是转调 `loop(ctx)`。
- `write()` `:234-262`：先写同名 `.tmp` 再 `renameTo`（同目录 rename 原子）。
  直接截断会让 JS 恰好读到空串、把那一周期当成「没连」，主题白跳一次；rename 失败退回直接写。
- 状态文件 `carlink_state.txt`：一行两值 `连接(0/1)|深浅(-1|0|1)`，落在 RNFS 的 `ExternalDirectoryPath`
  （`/sdcard/Android/data/<pkg>/files`）；消费方 `src/core/init/sunTheme.ts:13-31`，读不出/不存在一律
  `{connected:false, carMode:-1}`。
- 除 `ContentObserver` 之外全用反射，所以不需要 android.jar。

为什么单独一个 dex：主 dex 的 `method_ids` 已经贴着 65536 上限走，硬塞进去会炸；
实测整套注入在 classes.dex 上只净耗 **+1 个 method_id**（62667，余量 2868），而桥接代码要几百个，所以外置。
`$NightObserver` 必须有，否则车机深浅色不会秒级更新（第 [4b] 步也断言了）。
多 dex 靠加载顺序即可，**不需要 `MultiDex.install`**（`MainApplication` 只加了 1 行挂钩，没塞任何安装逻辑）。

---

## §A6 D 层：打包

- `AndroidManifest.xml` 是二进制 AXML，不能按文本改。第 [5] 步只调 `scripts/axml_rename.py` 做字符串池级替换，映射表 4 项：

  | 原串 | 新串 |
  | --- | --- |
  | `cn.toside.music.mobile` | 目标包名 |
  | `cn.toside.music.mobile.DYNAMIC_RECEIVER_NOT_EXPORTED_PERMISSION` | 同步换 |
  | `cn.toside.music.mobile.androidx-startup` | 同步换 |
  | `cn.toside.music.mobile.provider` | 同步换 |

  `.MainActivity` / `.MainApplication` 是类名，**不能动**（实测两成品的 `launchable-activity` 仍是
  `cn.toside.music.mobile.MainActivity`）。`axml_rename.py` 自带「原样重排逐字节自检」（`:48-49`）、
  要求 mapping 每项都在字符串池里（`:56-57`）、重写文件头长度（`:60`）并回读校验（`:61-62`）。
  `scripts/axml_label.py` **本条线完全不被调用** —— 它是旧线改应用名「洛雪音乐」用的；
  本线第 [8] 步反而断言 label 必须还是 `LX Music`（`rebuild191.py:618-620`）。同理
  `axml_rename.py:70-77` 里那组 `com.lxwalnut.*` 映射是旧线遗留，只有手工跑它的 `__main__` 才用到。
- `resources.arsc` 是另一套机制（第 [6] 步）：扫 chunk type `0x0200`，从 `off+12` 起在 256 字节槽位里就地写
  UTF-16LE 并补零，带 `assert len(new) <= 256`（`rebuild191.py:386-400`）；找不到 package 段直接报错。
- 第 [7] 步：重打包时**按类型**丢弃旧 v1 签名条目（`:411-418`，不能写死文件名，旧线踩过这坑）→
  `zipalign -f -p 4` → `apksigner sign --v1 true --v2 true --v3 false --v4 false`。
  于是 `META-INF/CERT.SF/.RSA` 被换成 `META-INF/ANDROIDD.SF/.RSA`。

---

## §A7 从零重做

### §A7.1 依赖

| 项 | 版本 | 来源 |
| --- | --- | --- |
| Python | 3.12（3.8+ 应可） | — |
| Java | Temurin 21 实测通过（脚本注释写 JRE 8+） | — |
| baksmali / smali | 2.5.2 | Maven Central `org.smali:baksmali` / `org.smali:smali`，配 `libs/`：dexlib2-2.5.2、util-2.5.2、antlr-runtime-3.5.2、guava-28.2-android、jcommander-1.48 |
| ecj | 3.21.0 | Maven Central `org.eclipse.jdt:ecj:3.21.0` |
| r8（内含 D8） | 1.6.84 | **不在 Maven Central（404）**，用 Google Maven：`https://dl.google.com/android/maven2/com/android/tools/r8/1.6.84/r8-1.6.84.jar` |
| build-tools | 34.0.0 | Android SDK；Windows 是 `zipalign.exe`/`aapt.exe`，Linux 同名不带后缀，脚本两边都试 |
| Node + 工程依赖 | 只在重编 bundle 时需要 | 上游工程里 `npm ci` |

把 jar/exe 放到 `scripts/tools/` 与 `scripts/bt/android-14/`（细节见 `scripts/工具与依赖.txt`）。
`android-debug.keystore` 工具包里已带（就是上游自带公签，别换）。

### §A7.2 走完整链路（含 JS 改动）

```bash
# 0) 取上游源码（用随本文交付的源码树就跳过 0) 和 1)，直接从 2) 开始）
git clone --depth 1 --branch v1.9.2 https://github.com/lyswhut/lx-music-mobile.git lx-music-mobile-1.9.2
cd lx-music-mobile-1.9.2

# 1) 打原包名那片补丁（随源码包交付，与 lx-music-mobile/ 同级；不在 git 仓库里 git apply 也能用，
#    等价写法：patch -p1 < ../lx-music-1.9.2-CarWith-src-原包名-cn.toside.music.mobile.patch）
git apply ../lx-music-1.9.2-CarWith-src-原包名-cn.toside.music.mobile.patch

# 2) 装依赖 + 现编原包名那份 bundle（只有这一份）
npm ci
PKG=cn.toside.music.mobile SRC191=<工程树> python3 scripts/bundle191.py

# 3) 出 APK：官方 lx-music-mobile-v1.9.2.apk 所在目录用 MOBILE_DIR 指定
#    ⚠ 跑之前先按 §B3.1 松掉两处旧 provider 断言，否则第 [8] 步必定 SystemExit
MOBILE_DIR=<目录> PKG=cn.toside.music.mobile VER=1.9.2 python3 scripts/rebuild191.py
```

### §A7.3 「不装 Node」那条捷径：原包名用不了

捷径本身是「把 `prebuilt/index-<包名>.android.bundle` 放进 `scripts/js191/`，跳过第 2 步」，
但它只对那两份现成 bundle 成立 —— `prebuilt/` 里没有 `cn.toside.music.mobile` 那份字节码，
而 provider authority 烤在 bundle 里（§A4），拿错包名的 bundle 会在 `rebuild191.py:67-72` 直接 `SystemExit`。
⇒ **本文口径必须装 Node**，按 §B3.2 的 B1 两步现编。

### §A7.4 换版本 / 换 donor 时要注意的

- **versionCode 79003 的来历**：`package.json:4` 的 `versionCode 79` × 1000 + ABI 序号（`arm64-v8a` = 3），
  规则在 `android/app/build.gradle:101 / :174 / :180`（`versionCodes = ["armeabi-v7a": 1, "x86": 2, "arm64-v8a": 3, "x86_64": 4]`）。实测官方与成品的 `lib/` 里都只有 `arm64-v8a`。
  ⇒ 换了 donor APK（比如 armeabi-v7a）就不再是 79003，别照抄。
- **react-native-track-player 的钉法**：实测工程 `package.json:67` 钉在 `#d76fe8ff…`。
  1.9.1→1.9.2 上游确实换过它（两个提交，只动 `MusicManager.java` 的音频卸载参数），
  而被挂钩的 `TrackMetadata`(l4/d) 与 `MetadataManager`(k4/b) 源码没动 —— 这才是这次能平移的真正原因。
- **不要以为「升级免费」是普遍规律**：说明书自己也提醒过，R8 的混淆名是整程序算出来的，
  下次换版本完全可能变。可行的认法是去 `classes.dex` 里找锚点串（本次实测官方 1.9.2：
  `lyricLine` 不存在、`android.media.metadata.LYRICS` 存在、`ucar.media.metadata` 不存在），
  而不是记住 `l4/d` 这个名字。
- **从说明书还原工具链**：`scripts/restore.py` 会把说明书里 `@@@FILE <路径> | bytes=N | sha256=…` 的围栏块
  还原成文件并逐块校验（字节数 + sha256，不过就退出；CRLF 先归一 LF）。说明书共 **18 个围栏块**，
  但 `tools/`、`bt/`、`android-debug.keystore`、`js191/` 二进制**不在其中**，必须手工补。
- **打包工具包这类目录时**：构建机上的 `bt/android-14` 如果是指向别处的符号链接，`cp` 要用 `-L` 解引用，
  否则对面拿到死链（构建机上实测：`bt/android-14` 是指向 `/tmp/bt/android-14` 的符号链接，`tools/` 实测 12M）。
- **上游源码 zip**：`bundle191.py` 需要它（默认在 `$MOBILE_DIR/lx-music-mobile-1.9.2.zip`，可用 `ZIP191=` 指路）。
  用 GitHub tag 页的 Source code (zip) 即可 —— 它解压后的顶层目录正好是 `lx-music-mobile-1.9.2/`，
  与脚本里 `TOP = 'lx-music-mobile-%s' % VER`（`bundle191.py:33`）对得上；换个文件名就会解不出源码。

---

## §A8 校验清单：脚本每一步在断言什么

`rebuild191.py` 是自校验的，任何一条不过就 `SystemExit`，不会给你出一个「看起来成功了」的 APK。

| 步 | 内容 | 拦的是什么 |
| --- | --- | --- |
| [0] | bundle 新鲜度：不比 `js4/` 和 `patch191_*_js.py` 旧（`:623-644`，可 `--allow-stale-bundle` 跳过，不建议） | 改了 JS 忘了重编 bundle |
| 入口预检 | bundle 里必须含 `<包名>.provider`（`:67-71`） | 拿错包名的 bundle —— 否则错误 APK 已经写好签好才被发现 |
| [1] | 从原版解出 `classes.dex` / `AndroidManifest.xml` / `resources.arsc`，打印条目数（官方实测 798） | — |
| [2] | baksmali 反汇编，`--debug-info false`，打印文件数（实测 9263） | — |
| [3] | 挂钩：8 处锚点各自命中次数必须 == 1（`:278-291`）；`_smali_path()` 同时兼容 `l4.1/d.smali` 与 `l4/d.smali`（`:262-275`） | 上游改了代码导致锚点漂移或多重命中；混淆目录改名 |
| [4] | smali 汇编 + 读 dex 头打印 method_ids/类数/剩余位，`>=65535` 直接停（`:294-312`），再查 3 个必需串 | 撞方法数上限（装上去就是 NoClassDefFoundError） |
| [4b] | classes2.dex（`:315-376`）：ecj + d8；断言必含 7 个串、**不得混进 `Landroid/` 或 `Lcom/facebook/`**、非 probe 时类集合**正好 3 个**、必须含 `CarLinkBridge$NightObserver`，再用 baksmali 反查 `start(Ljava/lang/Object;)V` | 编译桩被误打包；桥接缺类导致深浅色不秒级更新；方法签名写岔 |
| [4后] | dex 回读：挂钩描述符串确实存在于产物里（`:374`） | 描述符/签名写岔（写岔就是启动即 NoSuchMethodError 闪退） |
| [5][6] | 清单与 arsc 换包名（见§A6 节；arsc 有 256 长度约束） | 结构损坏 |
| [7] | 按类型丢旧 v1 签名 → `zipalign -f -p 4` → apksigner `--v1 true --v2 true --v3 false --v4 false` | 旧签名残留导致装不上；对齐不合 |
| [8] | 逐条回读成品（`:465-620`）：证书指纹 == 钉住的 SHA-256；`zipalign -c`；条目 CRC 三清单（新增/删除/变更）；bundle 必含 **15** 串、禁含 **18** 项、且不含本机路径；`classes.dex` 4 个串；**类集合与原版一致**（比集合不比顺序，类排列会随目录枚举顺序变）；`l4/d` 5 项 + `k4/b` 6 项计数；ucar 两个键写在 `setMetadata` 之前；开关门控成立；`MainApplication` 挂钩计数；`aapt dump badging` 的包名与应用名 | 换钥匙、烤进本机路径、丢类、门控写反、包名/标签串错 |

第 [8] 步里 `aapt` 读不了中文路径，脚本会先把成品拷成 `probe.apk` 再读（`:603-604`）—— 自己手工跑 `aapt`
时也注意别放在中文目录下。

跑完后建议再手工过三条（已实测，见 《改动面审计》）：
`aapt dump badging` 看 package；`apksigner verify --verbose`；`dexdump -f classes2.dex` 确认类集合恰好 3 个。

最常见的三种报错：① `找不到挂钩目标 …`（混淆类名变了，按§A7 节的重定位办法改 `EDITS_BY_FILE` 的键）；
② `xxx 锚点不唯一(N)`（smali `:288`、JS 侧 `patch191_lyric_js.py:26` / `patch191_suntheme_js.py:131`，
上游文本漂移或驱动被重复跑）；③ bundle 不配套 / 不新鲜（缺 `<pkg>.provider`，或 [0] 报旧）。
次常见：证书指纹不等（换钥匙了）、classes2 类集合不符（桩类混进 dex）、method_ids 撞 65535。

---

## §A9 已知坑与口径纠正（务必读）

1. **裸 diff 会误报越界改动。** `diff -rq 官方smali vs 成品smali` = 25 个文件，其中 22 个是
   smali `assemble → baksmali` 往返产物：只是把 `.field ... = false` / `= null` 的显式初始化器抹掉
   （实测 96 行，100% 是 `.field` 行，`= false` 31 处、`= null` 17 处，非 field 改动 0 行），
   DEX 里缺省即零值，语义完全相同。**验证必须带 control 往返基线**（把官方 dex 同套路 assemble→baksmali），
   `control vs 成品` 才恰好 3 个文件。
   再补一条更好用的二次基线：`官方 1.9.2 vs 作者 1.9.1 成品树` = 27 个文件 = 22 往返噪声 + 3 注入 + 2 真实语义漂移
   （`j4/c.smali`、`T/D$b$a.smali`）。看懂这 27 的构成，基本就不会再被 diff 数字吓到了。
2. **`diff` 出来的标签编号会变，别当作改动。** 脚本里写的是自定义标签 `:cond_ucar` / `:cond_ucar_off` / `:cond_ucar_line`
   / `:cond_ucar_done`，汇编再反汇编后会被重编成 `:cond_27` 这类数字标签（实测 `smali_out192/k4/b.smali:1179`）。
   ⇒ **不要写「脚本的新串能在成品 dex 里逐字节搜到」**，只能查顺序与计数。比较时要按标签归一化
   （重编号 + 忽略空行与 `.line`）。已实测：这 3 个文件与作者 1.9.1 成品同名文件**逐字节相同**
   （md5：MainApplication `9f16a453…`、k4/b `053f461e…`、l4/d `7e4055ef…`）。
3. **bundle 与包名必须配套。** `APP_PROVIDER_NAME` 烤在 Hermes 字节码里，拿错包名的 bundle 会打出装不上的 APK。
   源码层唯一差异就是 `src/config/constant.ts` 那一行。
4. **同一份代码的 `classes.dex` 字节可以不同。** 差异只是 smali 2.5.2 assemble 时 string_ids/class_defs
   排列的不确定性；反汇编后逐文件比对 **0 个文件差异**，`classes2.dex` 则是完全同一份（md5 `1972dab8…`）。
   ⇒ 判断「这是哪个包」只能看 `AndroidManifest.xml` 的 package，不能靠 dex 哈希。
5. **「只改 3 个 smali」这个说法只对 classes.dex 的代码面成立。** APK 整包层面实际是：
   新增 `classes2.dex`、`META-INF/ANDROIDD.SF/.RSA`；删除 `META-INF/CERT.SF/.RSA`；
   变更 `classes.dex`、`assets/index.android.bundle`、`AndroidManifest.xml`、`resources.arsc`、`META-INF/MANIFEST.MF`。
6. **成品 `classes.dex` 比官方小 27836 字节是正常的**：`--debug-info false` 往返丢弃调试信息；
   `class_defs` 数量与官方一致（9263）说明没丢代码。
7. **桥接类的 Java 包名是历史命名 `com/salt/music/bridge/CarLinkBridge`，它不是 applicationId。**
   它只是 dex 里的一个类路径，`classes.dex` 的引用串与 `classes2.dex` 的类名必须逐字一致；
   原包名构建同样不用改（引用解析得开，不会 `NoClassDefFoundError`）。
   真要挪进 `cn/toside/music/mobile/bridge/`，得同时改四处：`native/java/CarLinkBridge.java` 的 `package` 行、
   注入 `MainApplication` 的那条引用串（本文附录 C.3）、`rebuild191.py` 里 classes2 必含类名断言、
   以及 `native/smali/diff/` 与 `native/smali/patched/` 两份参照件。
   JS 侧另有两个同批遗留的缓存 key `@salt_sun_location` / `@salt_sun_budget`（`src/utils/sunLocation.ts:26-27`），
   纯字符串、与包名无关；改名只会让老用户重新定位一次。
8. **别用 `apktool` 走这条路**：它会重排资源、换签名体系，改动面比上面这些大得多。
9. **读 DEX header 时 `0x58` 是 `method_ids_size`，`0x5c` 才是 offset**，读反会得到 610792 这种离谱数字。
10. **文件名拼法**：本版输出是「适配CarWith」；1.9.1 那条线的历史文件名是「适配CariWith」（多一个 i，当年打错）。
    两份放一起别混。
11. **`--probe` 探针模式在工具包里不可用**。它是 1.9.1 那条线遗留的诊断能力：开关在 `rebuild191.py:41`，
    会多编一个 `jsrc_probe` 源目录（`:323`）、给 classes2 多加 `DayNightProbe` / `carwith_daynight_probe.txt`
    两个必含串（`:342-347`）、在 `MainApplication` 的桥接挂钩后再插一行 `DayNightProbe;->start`（`:255-258`）、
    输出名加 `-探针` 后缀（`:54-59`），用途是在实机上找「车机切亮暗时手机上有什么跟着变」。
    **工具包里没有 `jsrc_probe/`，说明书附录也没附它的正文**，跑 `--probe` 会在 `:328` 报
    「找不到 jsrc_probe 下的 Java 源」；功能补丁本身不依赖它。
12. **别拿 sha256 当验收标准 —— `classes.dex` 字节级不可复现**。同一份 smali 两次 assemble，
    string_ids / class_defs 的排列可能不同（R8 名字是整程序算的，smali 2.5.2 也不保证顺序稳定）。
    实测：同代码的成品先后有两对哈希被写进过文档（`b4bdbe89…`/`c60600e8…` 见旧《原说明书》第十节，
    `c2eb4fec…`/`8f0f15ff…` 见工具包《APK哈希参考.txt》），两对从第 3010 字节起就不同 —— 差异只在 `classes.dex` 及其派生的
    `META-INF/MANIFEST.MF`（只差 classes.dex 那一行 `SHA-256-Digest`）、`.SF`、`.RSA`；
    大小同为 14265630，签名、zipalign、badging 全部复测通过，反汇编后语义零差异。
    ⇒ 验收要看**反汇编后的文件比对 + 类集合 + 计数 + 实机行为**，不要看整包哈希。
13. **JS 读、原生写是两条独立时钟**：原生轮询 `TICK_MS = 5000ms`（`:35`），JS 每 2000ms 读一次状态文件；
    原生只在值变化时才写。按两段时钟推算，「车机切深色后手机上跟着变」最坏约 5+2=7 秒，这是设计而非 bug。

---

## §A10 术语表

| 术语 | 含义 |
| --- | --- |
| CarWith / Jovi InCar | vivo 的车机互联协议；车机侧按 `ucar.media.metadata.*` 专用键显示歌词 |
| `l4/d` / `k4/b` / `s4/d` | 上游 release 包里的混淆类名，分别是 TrackMetadata / MetadataManager / MainApplication 的父类 |
| `:cond_*` / `:goto_*` | smali 里的分支与跳转标签，baksmali 会自动重编号 |
| control 基线 | 把官方 dex 用同一套工具 assemble→baksmali 往返出来的「零注入产物」，用来剥离管线噪声 |
| `ucZ` | 注入到 `k4/b` 的静态标记，记上次发布是否送过 `lyricLine`，用来只在关开关那一刻清一次 ucar 键 |
| `ucar_night_mode` | 车机写进 `Settings.Global` 的深浅色信号，0=浅色、1=深色 |
| `carlink_state.txt` | 原生桥接落给 JS 的状态文件，格式 `连着(0/1)|深浅(-1|0|1)`，JS 每 2 秒读一次 |
| Hermes / hermesc / bundle | RN 的 JS 引擎、其字节码编译器、以及编译出的 JS 产物（provider authority 烤在里面 ⇒ 一个包名一份） |
| 安卓公签 | 上游自带的 `debug.keystore`，证书 SHA-256 已钉进脚本，保证可覆盖安装 |

---

## §A11 关键文件与校验值（工具包视角）

完整清单见工具包内的 `SHA256SUMS.txt`（源码包另有一份，只覆盖源码树那 758 个文件与那片原包名补丁）。下面是复核时最常要用到的（sha256 前 16 位，均为实测）：

| 文件 | sha256(16) |
| --- | --- |
| `…src-原包名-cn.toside.music.mobile.patch`（随源码包交付，1104 行 / 22 文件） | `26067febb82ce7ee` |
| `scripts/rebuild191.py`（659 行） | `f858de4e6e0f7121` |
| `scripts/bundle191.py`（176 行） | `1df38351b4ca0068` |
| `scripts/patch191_lyric_js.py`（243 行） | `e822d17a5d9edd06` |
| `scripts/patch191_suntheme_js.py`（156 行） | `45a31e92e6b38fa8` |
| `scripts/restore.py` | `87eda2eed96bdc2d` |
| `scripts/axml_rename.py` / `axml_label.py` | `dbb3f18f828a7ff0` / `ce27282ff12834e8` |
| `scripts/check_lang_params.py` / `measure_text_width.py` | `bf4a00eb408fe8bb` / `67fcfb89bca18444` |
| `scripts/android-debug.keystore`（2257 B） | `221e0a3106aa4c3c` |
| `native/java/CarLinkBridge.java`（282 行） | `ed3578886e653bb1` |
| `native/java/ContentObserver.java`（16 行） | `50a76e9d1d5b60e0` |
| `prebuilt/classes2.dex`（遗留成品 APK 内的同一份，md5 `1972dab8…`） | `727c0c2252d13cec` |

原包名的 bundle 没有现成件（`prebuilt/` 那两份都不是），所以要现编 —— 见 §A4 的 ⚠ 与 §B3.2。

三条已经实测过的等价性，改东西前先确认它们还成立：

1. 原包名这片补丁 1104 行 / 22 文件 / +788 −8；工具包 `patches/` 里那两份各多一行（`APP_PROVIDER_NAME`）
   ＝ 1117 行 / 23 文件 / +789 −9，其余逐字相同（已 diff 验证）。本文一律用前者。
2. 源码补丁在纯上游树上 `git apply` 后与 `source-modified/` **`diff -r` 零差异**（往返验证过）。
3. `native/smali/patched/` 三个文件与交付 APK 里的 dex 反汇编结果一致，且与作者 1.9.1 成品
   **逐字节相同**（md5：MainApplication `9f16a453…`、k4/b `053f461e…`、l4/d `7e4055ef…`）。

成品 APK 本身的 sha256 只用于确认「拿到的是哪个文件」，**不要当验收标准**（原因见§A9 节第 12 条）。

---

# 第二部分 重置：重做、逐层撤销、验证与升级

## §B0 一分钟版

```bash
UP=/tmp/up     # 官方 v1.9.2 源码树，见 §B2.3 的取法
TREE=<本源码树根目录>

# 1) 这棵树相对官方改了什么（22 个文件，全在 src/ 内，+788/-8）
diff -rq  --exclude=.git --exclude=node_modules $UP $TREE | wc -l                                       # 22 = 15 处修改 + 7 个新增
diff -ruN --exclude=.git --exclude=node_modules $UP $TREE | grep '^+' | grep -v '^+++' | wc -l           # 788
diff -ruN --exclude=.git --exclude=node_modules $UP $TREE | grep '^-' | grep -v '^---'  | wc -l           # 8

# 2) 改完之后必须重编 bundle，否则手机上跑的还是旧字节码（§B3.2）

# 3) 想整套撤销回官方：直接用 $UP 覆盖 src/ 再删掉这 7 个新增文件
rm $TREE/src/core/init/sunTheme.ts $TREE/src/utils/sunState.ts $TREE/src/utils/sunrise.ts    $TREE/src/utils/sunLocation.ts $TREE/src/screens/Home/Views/Setting/settings/Player/IsShowCarWithLyric.tsx    $TREE/src/screens/Home/Views/Setting/settings/Theme/FollowCarWithTheme.tsx    $TREE/src/screens/Home/Views/Setting/settings/Theme/FollowSunTheme.tsx
cp -a $UP/src/. $TREE/src/
```

> 上面三条里有两个坑：`diff` 必须带 `-N`（不带时新增文件只出现 `Only in` 行，行数会少掉一大半），
> 剔文件头要写 `grep -v '^+++'`（写成 `'^\+\+\+'` 在 BRE 里是量词，会把所有行都当成头行剔光）。
> 统计前先 `--exclude=node_modules`：装过依赖的官方树会多出上万行噪声。

出 APK 需要官方 `lx-music-mobile-v1.9.2.apk` + Node + 一套 jar/exe，见 §B3。
源码树只覆盖 A 层：B/C/D 三层不在源码里，在 APK 与工具包里（§B1）。

---

## §B1 这套改动分四层，每层的重置手段不一样

| 层 | 载体 | 在哪 | 正向（重做） | 重置（撤销） | 验证 |
| --- | --- | --- | --- | --- | --- |
| A 源码 | `src/**.ts(x)` / `src/lang/*.json` | **就在本源码树里**（唯一的源码层） | `git apply …原包名….patch`（随源码包交付） | `git apply -R` 同一片补丁 | §B0 三条 diff 统计 = 22 文件 / +788 / −8 |
| B bundle | `assets/index.android.bundle`（Hermes 字节码） | 不在源码树里（生成物） | Metro + hermesc，或 `scripts/bundle191.py` | 用原版 APK 的 bundle 覆盖回去 | 15 条必含串 / 18 条禁串（本文 §B5.3） |
| C dex | `classes.dex` 改 3 处、新增 `classes2.dex` | 不在源码树里；native 真源码见本文附录 A、C | `scripts/rebuild191.py` 第 [2]-[4b] 步 | 从原版 APK 抽回 `classes.dex`、删掉 `classes2.dex` | `method_ids` 62666 → 62667；`classes2.dex` 恰好 3 个 class_def |
| D 打包 | `AndroidManifest.xml`(AXML)、`resources.arsc`、签名 | 不在源码树里，纯二进制 | 第 [5]-[7] 步（axml_rename + 重打 + zipalign + v1/v2 签） | 原封不动地用原版 APK 的清单与 arsc | `aapt dump badging` 的 package 名、`apksigner verify` 指纹 |

**最容易搞错的一点**：A 层改的是 `.ts/.tsx`，手机跑的却是 B 层的字节码。
只回退 A 层不回退 B 层，功能照样生效；反过来只回退 A 层就以为「已经撤销干净」，是假的撤销。
源码树里那片补丁管不到 B/C/D 三层，那三层永远以「你手里那个 APK 的条目内容」为准。

---

## §B2 前置检查（做任何一条路之前先跑）

### §B2.1 环境与工具

| 项 | 要求 | 自检命令 | 备注 |
| --- | --- | --- | --- |
| git | ≥ 2.25 | `git --version` | 只有要用 git 管这棵树时才需要；反向补丁自检要 |
| Python | 3.8+（实测 3.12.13） | `python3 -V` | 脚本只用标准库 |
| Java | 8+（实测 Temurin 21） | `java -version` | 只有 baksmali/smali/ecj/r8/apksigner 要 |
| Node + 工程依赖 | 只在重编 bundle 时要 | `node -v` | 必须 `npm ci` 出**非空真目录**的 `node_modules`，junction/联接点不行（本文 §A4 第 3 条） |
| Android build-tools | 34.0.0 | `ls bt/android-14/zipalign` | zipalign / aapt / lib/apksigner.jar |
| baksmali+smali | 2.5.2 + libs 5 个 jar | `ls tools/baksmali.jar` | Maven Central `org.smali:*` |
| ecj | 3.21.0 | `ls tools/ecj-3.21.0.jar` | Maven Central `org.eclipse.jdt:ecj:3.21.0` |
| r8（含 D8） | 1.6.84 | `ls tools/r8-1.6.84.jar` | **Maven Central 没有（404）**，从 Google Maven 取（URL 见 《工具与依赖》） |
| 官方 APK | `lx-music-mobile-v1.9.2.apk` | 文件名与 `VER` 对上 | 走 B/C/D 层必须有它，且必须是官方签名那版 |

jar/exe 与 `MOBILE_DIR`、`tools/`、`bt/` 的摆放位置见 《工具与依赖》，那份是权威清单。

### §B2.2 基线：这棵树锚在哪个上游提交

```bash
git rev-parse v1.9.2^{commit}    # 期望 0669d1b6d419967822ff52ec06e50b9613bfd136
diff -rq  --exclude=.git --exclude=node_modules /tmp/up <源码树> | wc -l     # 期望 22（15 differ + 7 only-in）
diff -ruN --exclude=.git --exclude=node_modules /tmp/up <源码树> | grep '^+' | grep -v '^+++' | wc -l   # 期望 788
```

- 上游仓库：`https://github.com/lyswhut/lx-music-mobile`，基线 tag `v1.9.2`。
  `v1.9.2` 是**附注标签**：标签对象 `d1916db5de11359886db84517ad07f60e1901700` → 指向提交 `0669d1b6…`。
  脚本或 CI 里要拿提交号，必须写 `v1.9.2^{commit}`，直接 `rev-parse v1.9.2` 拿到的是标签对象。
- 源码树 = 上面那个提交 + 22 处 `src/` 改动，没有第二手改动。
  ⇒ 认锚点用内容，不用提交号：§B2.3 那条 `diff -rq` 恒为 15 differ + 7 only-in = 22，
    行数恒为 +788/−8；多一条就是这棵树与本文不同源。
- 源码树里 `src/config/constant.ts:98` **保持上游原值** `APP_PROVIDER_NAME = 'cn.toside.music.mobile.provider'`。
  改包名是构建期由脚本替换那行，不是源码状态（原因：provider 串烤进 Hermes 字节码，见本文 §A4）。

### §B2.3 动手前的完整性自检（两步就够）

```bash
# ① 取官方基线（本文所有「和官方比」的检查都以它为准）
git clone --depth 1 --branch v1.9.2 https://github.com/lyswhut/lx-music-mobile.git /tmp/up
# 期望 HEAD = 0669d1b6d419967822ff52ec06e50b9613bfd136

# ② 整树对比：本源码树相对官方只应有那 22 个文件的差异，且全在 src/ 里
diff -rq --exclude=.git --exclude=node_modules /tmp/up <源码树>
# 期望：15 行 "Files ... differ" + 7 行 "Only in <源码树>: ..."（新增的 7 个文件）
```

`diff -rq` 的输出条数本身就是改动面：15 处修改 + 7 处新增 = 22。多一条就说明这棵树与本文不同源。
文件总数同样对得上：官方 751 个，源码树 758 个（差的就是那 7 个新增文件）。
本人在出这份源码时实跑过：结果正是 15 differ + 7 only-in，且 `src/config/constant.ts` 不在其中（provider 保持官方原值）。
想把改动做成一片可复用的补丁：在**两棵树的公共父目录**里跑 `diff -ruN --exclude=.git --exclude=node_modules 官方树 改动树 > carwith-v1.9.2.patch`（必须相对路径，`patch -p1` 才对得上），
反向撤销就是 `patch -p1 -R < carwith-v1.9.2.patch`（工具包 `patches/` 里也有现成的三份，同一内容）。

---

## §B3 从源码重做出 APK

完整链路的权威步骤是本文 §A7，这里只给「命令 + 期望输出 + 失败含义」的重做口径。
第 [0]-[8] 这些方括号标签是 `rebuild191.py` 自己打印的步骤号，照它对答案最省事。

### §B3.1 包名固定为上游原值，先解决 bundle 与两处断言

| 目标包名 | 源码状态 | bundle 怎么来 | 需要改 AXML/arsc 吗 | 与官方包的关系 |
| --- | --- | --- | --- | --- |
| `cn.toside.music.mobile`（本文唯一口径） | 直接用本源码树，不改一行（`constant.ts:98` 保持原值） | **只能现编**（§B3.2 B1 两步） | 不需要，映射表变成恒等 | 同包名同签名 ⇒ 覆盖安装官方包；**不能与官方共存** |

包名和 bundle 是**一对一钉死**的：`rebuild191.py` 在开工前就验 `<PKG>.provider` 在不在字节码里，
不在直接 `SystemExit`（`rebuild191.py:67-72`）。这不是误报，别绕过。

**走原包名（`cn.toside.music.mobile`）必须先松两处断言**：脚本默认 applicationId 会被换掉，
第 [8] 步的禁串表第一条就是旧 provider `cn.toside.music.mobile.provider`（`rebuild191.py:501`），
`bundle191.py:168` 的残留检查里也有同一个串。而原包名 bundle **必须**含这个串才能过必含检查 ——
两条直接对撞，不改就必定 `SystemExit`。改法（二选一；改完这份脚本就只服务原包名，别再拿它出换包名的包）：
把该串从两个表里摘掉，或包一层条件（例如
`gone = [x for x in gone if not (OLD_PACKAGE == NEW_PACKAGE and x.endswith(b'.provider'))]`）。

### §B3.2 重编 bundle（两种做法，别混着用）

**B1 · 手工两步（推荐：本源码树已经是打过 A 层补丁的状态，不需要 git）**
再强调一次：本源码树的 `src/` 已经是打过 A 层补丁、且 provider 保持原值的状态，所以**不要再跑 `patch191_*_js.py`**
（那两个驱动是「锚点唯一 + 命中一次」式插入，在已打补丁的树上重跑会因为重复插入报错，
这是保护，不是 bug）。直接：

```bash
cd <源码树根目录>
npm ci                                   # node_modules 必须是非空真目录，junction 不行
mkdir -p js191
node node_modules/@react-native-community/cli/build/bin.js bundle \
     --platform android --dev false --entry-file index.js \
     --bundle-output js191/index-cn.toside.music.mobile.android.bundle

cd js191                                 # hermesc 必须 cd 进输出目录、用相对文件名调用
../node_modules/react-native/sdks/hermesc/linux64-bin/hermesc -emit-binary \
     -max-diagnostic-width=80 -out index.hbc index-cn.toside.music.mobile.android.bundle
mv -f index.hbc index-cn.toside.music.mobile.android.bundle
```

Windows 用 `node_modules\react-native\sdks\hermesc\win64-bin\hermesc.exe`，macOS 用 `osx-bin/hermesc`；
`HERMESC` 环境变量可显式指定。本文口径不动 `constant.ts`；只有确实要换 applicationId 时才改那一行
（唯一与包名绑定的源码行），再走同样两步。

**B2 · 脚本优先（`bundle191.py`）**
它会把源码树**恢复成上游源码 zip 里那份 pristine**，再按 `PKG` 跑两个驱动、Metro、hermesc，
产物写成 `js191/index-<PKG>.android.bundle`（`--pkg` 只编一份，`--keep` 不清树当前状态重编）：

```bash
SRC191=<工程树> ZIP191=<上游源码 zip：lx-music-mobile-1.9.2.zip> \
PKG=<包名> python3 scripts/bundle191.py --pkg <包名>
```

⚠ 不加 `--keep` 时它会 `rmtree(<tree>/src)` 再从 zip 解回来 —— **直接拿交付的源码树跑会把你的
`src/` 换成 zip 里那份 pristine**（zip 必须与 `v1.9.2` 同源，否则源码树与构建树就开始漂移）。
要么另建一棵专用构建树，要么跑完用 §B4.2 的反向复原把 `src/` 装回来，并把 zip 当成基线来管。

**bundle 新旧闸门**：`rebuild191.py` 第 [0] 步拿 `scripts/js4/*` 与 `patch191_*_js.py` 的 mtime
和 bundle 的 mtime 比，源更新就 `SystemExit`（防止「驱动是新的、成品是旧的」）。
从 APK 抽现成 bundle 时一般没问题；一旦你动过 `js4/` 或驱动脚本，要么重编 bundle，
要么明确加 `--allow-stale-bundle`。

### §B3.3 出 APK

```bash
MOBILE_DIR=<放着官方 lx-music-mobile-v1.9.2.apk 的目录> \
PKG=<包名> VER=1.9.2 \
python3 scripts/rebuild191.py
```

期望按顺序看到 `[0] bundle 不比 JS 源旧` → `[1] 从原版解包` → `[2] 反汇编 classes.dex（不含调试信息）` →
`[3] 挂钩` → `[4] 汇编 classes.dex` → `[4b] 编 classes2.dex（只桥接）` → `[5] AndroidManifest.xml 换包名` →
`[6] resources.arsc 换包名` → `[7] 重打包 + zipalign + 签名` → `[8] 校验`，最后 `[8]` 里 15 条
`bundle 含 … : True` 全 True、签名与对齐断言通过，输出文件名落在 `$MOBILE_DIR`。
任何一条不过脚本就退出，不会给你一个「看着成功」的 APK。

原包名（`PKG=cn.toside.music.mobile`）这条线**实际没有构建过**（手上只有遗留的那两份成品），
打包侧没有障碍：`MANIFEST_MAP` 四项映射变成恒等、arsc 替换字符串相同、
`DIST_NAME` 走默认分支 `lx-music-mobile-v1.9.2-车机歌词-cn.toside.music.mobile.apk`。
唯一的拦路就是 §B3.1 末尾那两处旧 provider 禁串 —— 先按 §B3.1 改掉再跑，
并且第一次跑要按§B5逐条对答案（不要只看脚本没报错）。

### §B3.4 「从 APK 抽 bundle」那条捷径：原包名不适用

对象是遗留的那两份成品 APK（抽出来的 bundle 与各自包名配套）：

```bash
mkdir -p scripts/js191
unzip -p <那条线的成品>.apk assets/index.android.bundle \
    > scripts/js191/index-<那个包名>.android.bundle
MOBILE_DIR=<官方APK目录> PKG=<那个包名> VER=1.9.2 python3 scripts/rebuild191.py
```

**原包名没有这种捷径**：任何现成 bundle 里的 provider 都不是 `cn.toside.music.mobile.provider`，
`rebuild191.py:67-72` 会当场 `SystemExit`。⇒ 原包名一律走 §B3.2 B1 现编。

产出与交付件**同代码**，但 `classes.dex` 的字节可能不同：smali 2.5.2 重汇编时 `string_ids` / `class_defs`
排列顺序不确定。所以 《APK 哈希参考》 里那两个 sha256 只能证明「拿到的是同一份文件」，
**不能**当重做正确性的验收标准（论证见 《改动面审计》§12）。

---

## §B4 逐层重置（撤销）

按「撤销半径」从小到大。每条都给出**做完之后应该看到什么**。

> 本节写的是 git 命令，但**这棵树不需要是 git 仓库**（随本文交付的纯净源码树就不是）。
> 手里只有纯树时按这两条替换：
> `git checkout v1.9.2 -- <path>` → `cp -a /tmp/up/<path> <源码树>/<path>`；
> `git apply -R patches/<补丁>` → `patch -p1 -R < patches/<补丁>`（`/tmp/up` 见 §B2.3）。
> 新增的那 7 个文件 git 与不 git 都得手工 `rm`。

### §B4.1 只退 A 层的一个功能线

- 退歌词线（保留主题跟随）：还原 `src/core/init/player/lyric.ts`、`src/plugins/player/utils.ts`、
  `src/plugins/player/playList.ts`、`src/core/lyric.ts`、`.../Player/index.tsx`、
  `.../Player/IsShowBluetoothLyric.tsx`，删掉 `.../Player/IsShowCarWithLyric.tsx`，
  并把 `defaultSetting.ts` 的 `player.isShowCarWithLyric`、`app_setting.d.ts` 的同名声明
  与 3 份语言包的 `setting_play_show_carwith_lyric` 手工删掉。
  ```bash
  git checkout v1.9.2 -- src/core/init/player/lyric.ts src/plugins/player/utils.ts \
      src/plugins/player/playList.ts src/core/lyric.ts \
      src/screens/Home/Views/Setting/settings/Player/index.tsx \
      src/screens/Home/Views/Setting/settings/Player/IsShowBluetoothLyric.tsx
  rm src/screens/Home/Views/Setting/settings/Player/IsShowCarWithLyric.tsx
  ```
  然后**必须**重编 bundle，否则手机上行为不变。
  ⚠ 歌词线与主题线在 `defaultSetting.ts` / `app_setting.d.ts` / 三份语言包 / `init/index.ts` 上是**交叉**的，
  整体 `git checkout v1.9.2 --` 这些文件会顺带把主题线也退掉。

- 退主题线：
  ```bash
  git checkout v1.9.2 -- src/theme/themes/index.ts src/core/init/index.ts \
      src/screens/Home/Views/Setting/settings/Theme/index.tsx
  rm src/core/init/sunTheme.ts src/utils/sunState.ts src/utils/sunrise.ts src/utils/sunLocation.ts \
     src/screens/Home/Views/Setting/settings/Theme/FollowCarWithTheme.tsx \
     src/screens/Home/Views/Setting/settings/Theme/FollowSunTheme.tsx
  ```
  `src/plugins/sync/client/client.ts`（那条线顺手删掉的 `toast('Sync connected')`）与本功能无关，
  想保持最小改动面就单独还原它：`git checkout v1.9.2 -- src/plugins/sync/client/client.ts`。

**验证退干净了**（三条都应无输出）：

```bash
grep -rl "isShowCarWithLyric" src | grep -v node_modules
grep -rl "followCarWithTheme\|followSunTheme" src
grep -rl "lyricLine" src
```

### §B4.2 A 层整体重置

```bash
# 原包名那份补丁就在源码压缩包里（与 lx-music-mobile/ 同级）
git apply -R lx-music-1.9.2-CarWith-src-原包名-cn.toside.music.mobile.patch
rm -f <§B4.1 那 7 个新增文件>     # 反向补丁只还原内容、不删新增文件
git status --porcelain           # 期望：空输出（这棵树除 src/ 外没有别的改动）
```

纯树写法（无 `.git`）最省事的一条就是 §B0 第 3 步：删掉 7 个新增文件，再 `cp -a /tmp/up/src/. <源码树>/src/`。
工具包 `patches/` 里那两份多 `constant.ts` 一行，反打完还要把它换回官方版才是原包名树。

`git apply -R` 报 `patch does not apply` 十有八九是补丁与树不匹配（例如拿那条线的补丁去退原包名树）。
先 `git apply -R --check`，或直接 `git checkout v1.9.2 -- src` 一步到位。

### §B4.3 git 层重置（只有把这棵树放进了 git 才用得上）

```bash
git reset --hard v1.9.2       # 丢掉本地提交，工作树一起回到基线
git clean -fdxn               # 连被 .gitignore 忽略的 node_modules 一起清（-x 慎用）
git revert HEAD               # 提交已经被别人取走过时用 revert，别 reset --hard
```

判断该用哪个：只有自己看 → reset；已经推出去、别人可能拉过 → revert（用正向新提交抵消）。
没建 git 仓库的话，本节全部等价于「把 §B4.1/§B4.2 的反向补丁再执行一遍」。

### §B4.4 B/C/D 层重置（APK 侧）

这三层没有「撤销脚本」，撤销办法就是**换回原版的对应条目**，或干脆从原版 APK 重做一遍。

```bash
# 看你现在这个 APK 被改过哪些条目
unzip -Z1 成品.apk | wc -l                      # 条目数：官方 1.9.2 = 798，改动后 = 799
mkdir x && cd x && unzip -q <官方>.apk && unzip -q -o <成品>.apk
sha256sum classes.dex classes2.dex assets/index.android.bundle AndroidManifest.xml resources.arsc
```

- 退 C 层：把官方 APK 的 `classes.dex` 塞回去，**删掉 `classes2.dex`**。
  只删 `classes2.dex` 不还原 `classes.dex` 会留下「挂钩指向不存在的类」→ 启动即 `NoClassDefFoundError` 闪退。
- 退 B 层：`assets/index.android.bundle` 换回官方那份。
  若 `classes.dex` 仍带着注入而 bundle 换成官方版，`lyricLine` 永远读不到（`getString` 返回 null），
  车机歌词静默不显示，不崩 —— 这正是《改动面审计》里「bundle 与 dex 是两个独立改动面」的含义。
- 退 D 层：`AndroidManifest.xml` / `resources.arsc` 换回官方那份，包名就回到 `cn.toside.music.mobile`；
  签名无论如何都要**重签**（改任何条目都会让 v2 签名失效），且必须继续用那把调试公钥，见 §B4.5。

重签一步都不能省：

```bash
zipalign -f -p 4 in.apk aligned.apk
java -jar bt/android-14/lib/apksigner.jar sign \
     --ks scripts/android-debug.keystore --ks-key-alias androiddebugkey --ks-pass pass:android \
     --v1-signing-enabled true --v2-signing-enabled true \
     --v3-signing-enabled false --v4-signing-enabled false aligned.apk
```

### §B4.5 签名与「装不上」的关系（重置时最容易踩）

证书指纹钉死为 `fac61745dc0903786fb9ede62a962b399f7348f0bb6f899b8332667591033b9c`
（= 上游源码自带的 `android/app/debug.keystore`，工具包里已带）。

```bash
java -jar bt/android-14/lib/apksigner.jar verify --print-certs 成品.apk
```

- 指纹一致 → 可以覆盖安装同包名的旧版本，歌单/音源/设置都在。
- 换了别的 keystore → 装同包名版本时必须先卸载，**数据全丢**。apksigner 照样报「verify 通过」，
  所以脚本第 [8] 步专门把指纹钉住，就是这个原因。
- 本文口径与官方**同包名**，所以改动版与官方版互斥：覆盖安装要求同签名，不存在「装两个」这条路。

### §B4.6 彻底回到「什么都没发生过」

```bash
rm -rf <成品>.apk                     # 手头的构建产物
git apply -R <那片补丁> && rm -f <上面列出的 7 个新增文件>   # 无 git：cp -a /tmp/up/src/. <源码树>/src/
adb uninstall <包名>                  # 手机侧
```
手机侧只需 `adb uninstall`，配置数据在应用私有目录里一起没了；
`/sdcard/Android/data/<包名>/files/carlink_state.txt` 由 native 线程写，卸载后不残留。

---

## §B5 验证矩阵（拿命令对答案）

> 本节三张表已在**遗留的那两份成品 APK**（同代码同签名，只有 applicationId 不同）上逐条复跑过：15 条必含全在、
> 18 条禁串全无、4 条本机路径泄漏全无、dex 侧 4 条串全 True、条目数 799。
> 也就是说这些期望值是实测数，不是推测值；只有官方侧基线那几条（798 条目、`method_ids` 62666、
> versionCode 79003）是引自 《改动面审计》 与 本文 §A7.4，
> 需要你自己手里那份官方 `lx-music-mobile-v1.9.2.apk` 才能复验。

### §B5.1 源码层

| 检查 | 命令 | 期望 |
| --- | --- | --- |
| 改动文件数 | `diff -rq --exclude=.git --exclude=node_modules /tmp/up <源码树> \| wc -l` | `22`（15 differ + 7 only-in；含 `constant.ts` 的那类补丁则为 23） |
| 增删行数 | `diff -ruN --exclude=.git --exclude=node_modules /tmp/up <源码树> \| grep '^+' \| grep -v '^+++' \| wc -l`（删除行同理换 '^-' 与 '^---'） | `788` / `8`（= `diffstat` 的 `22 files changed, 788 insertions(+), 8 deletions(-)`） |
| 新增文件 | `diff -rq --exclude=.git --exclude=node_modules /tmp/up <源码树> \| grep '^Only in <源码树>'` | 7 个：`core/init/sunTheme.ts`、`utils/{sunState,sunrise,sunLocation}.ts`、`…/Theme/Follow{CarWithTheme,SunTheme}.tsx`、`…/Player/IsShowCarWithLyric.tsx` |
| provider 未被改 | `diff -u /tmp/up/src/config/constant.ts <源码树>/src/config/constant.ts` | 空 |
| 三个设置键 | `grep -c "isShowCarWithLyric\|followCarWithTheme\|followSunTheme" src/config/defaultSetting.ts` | `3`；`src/types/app_setting.d.ts` 同样 `3`（歌词键在 `:209`） |
| 语言包新增条数 | `for f in zh-cn zh-tw en-us; do diff /tmp/up/src/lang/$f.json <源码树>/src/lang/$f.json \| grep -c '^>'; done` | 三份各 `10`（1 条歌词开关 + 9 条主题与状态文案），共 30 条，**没有删改任何原有条目**（`grep -c '^<'` 应为 0） |
| 语言包对齐 | `python3 scripts/check_lang_params.py <工程树>` | 无输出、exit 0（实参占位符不一致会 exit 1） |
| 文案宽度 | `python3 scripts/measure_text_width.py` | 12sp/310dp 与 14sp/296dp 两条闸门都不超 |

### §B5.2 dex / native 层

| 检查 | 命令 | 期望 |
| --- | --- | --- |
| 注入指纹计数 | `grep -c <串> <smali 文件>`（对反汇编出来的 3 个文件，或直接与 `native/smali/patched/` 对比） | `l4/d.smali`：`ucar.media.metadata` 2 次、`lyricLine` 1 次、新增实例字段 `j:Ljava/lang/String;`；`k4/b.smali`：`ucar.media.metadata` 4 次、`ucZ` 4 次、`lyricLine` 1 次、目标方法 `.registers` 由 5 变 6；`MainApplication.smali`：`CarLinkBridge` 1 次（`invoke-super {p0}, Ls4/d;->onCreate()V` 之后紧跟一行挂钩） |
| 官方树应为 0 | `grep -c ucar native/smali/official/**` | 全 0（`ucar` 只可能来自注入） |
| 挂钩总数 8 处 | `rebuild191.py` 里 `EDITS_L4D`(第 159 行, 4 项) / `EDITS_K4B`(第 194 行, 3 项) / `MAIN_APP_EDITS`(第 252 行, 1 项)，每项都断言锚点在文件中命中恰好 1 次 | 4+3+1=8；`native/smali/diff/` 三个 patch 是 6/7/1 个 **hunk**（一处挂钩可拆成多个 hunk，别把 hunk 数当挂钩数） |
| smali 三文件 | `diff -u native/smali/official/<f> <你反汇编的 f>` | 只看注入的几处差异；**自定义标签 `:cond_ucar*` 汇编后会被重编号成数字**（如 `:cond_27`），拿 `:cond_ucar` 去成品 dex 里逐字搜是错的验证法 |
| method_ids | Python 读 `classes.dex` 偏移 `0x5c` 处的计数 | 官方 1.9.2 = 62666；成品 = 62667（+1，余量 2868）。注意 `0x58` 是**大小**不是数量 |
| classes2.dex 类数 | `dexdump classes2.dex \| grep -c "Class descriptor"` | 恰好 3：`CarLinkBridge`、`$1`、`$NightObserver`；**不得**出现 `Landroid/database/ContentObserver;`（那是编译期桩类） |
| 无 MultiDex | `grep -c MultiDex smali_out/.../MainApplication.smali` | 0（只加了 1 行 `CarLinkBridge->start` 挂钩） |

### §B5.3 bundle / APK 层

| 检查 | 期望 |
| --- | --- |
| 15 条必含串（`rebuild191.py:491-496`） | 全 True。含 `player.isShowCarWithLyric`、`setting_play_show_carwith_lyric`、`common.followCarWithTheme`、`common.followSunTheme`、`isShowBluetoothLyric`、`ipinfo.io`、`<PKG>.provider`、3 条开关文案、5 条状态文案（定位成功/上次定位/定位失败/默认定位/极圈地区）。**中文串要按 UTF-8 与 UTF-16LE 两种编码各试一次**，只按 UTF-8 搜会假报缺失 |
| 18 条禁串（`rebuild191.py:501-516`） | 全部不出现：旧 provider `cn.toside.music.mobile.provider`（换包名那条线才成立；走原包名见 §B3.1 的改动）、`Sync connected`、9 组旧文案 × UTF-8/UTF-16LE 两种编码。另有 4 条本机路径泄漏检查（`:519-523`）：`:\Users\`、`:\\Users\\`、`补丁重打工具` ×2 编码 —— 命中就是 hermesc 被用绝对路径调了 |
| APK 条目数 | `unzip -Z1 <apk> \| wc -l`：官方 1.9.2 = 798，成品 = **799**（只多 `classes2.dex`；用 `unzip -l \| wc -l` 会把表头表尾也算进去） |
| 与官方有差异的条目 | 第 [8] 步会直接打印三个清单。期望：差异只有 `classes.dex`、`assets/index.android.bundle`、`AndroidManifest.xml`、`resources.arsc` + 签名三件 `META-INF/MANIFEST.MF`、`META-INF/*.SF`、`META-INF/*.RSA`；新增只有 `classes2.dex`；缺失为空 |
| dex 侧 4 条串 | 第 [8] 步还会打 `dex 含 lyricLine / ucar.media.metadata.LYRICS_WHOLE / ucar.media.metadata.LYRICS_LINE / android.media.metadata.LYRICS`，前三条 True、第四条也 True（原版本来就有 `android.media.metadata.LYRICS`，它是定位锚点的特征串，不是改动） |
| 类集合 | 第 [8] 步比对的是 **类名集合**而非字节：这个 dex 里有 1434 组只差大小写的类名（`Ld0/T;` 与 `Ld0/t;`），Windows 上 baksmali 靠加后缀避开撞名、排列还会随目录枚举顺序变，所以只比体积和字符串查不出丢类 |
| 对齐 | `zipalign -c -p 4 成品.apk` exit 0 |
| 签名 | `apksigner verify` 通过、v1+v2 开、指纹 = `fac61745…33b9c` |
| 包名/版本 | `aapt dump badging`：package **必须是 `cn.toside.music.mobile`**（本文口径，不该出现任何别的值）；versionCode 官方 1.9.2 为 79003（= `package.json` 的 79 × 1000 + arm64-v8a 序号 3；换 ABI 的 donor 就不是这个数） |

`aapt` 在含中文/空格的路径上会翻车，先 `cd` 到 ASCII 目录再传相对路径（本文 §A9 有记录）。

### §B5.4 实机验收（只有装上车才算完）

1. 连 CarWith，手机 `settings get global ucar_night_mode` 应为 `0` 或 `1`（`-1` = 没装/读不到）。
2. `cat /sdcard/Android/data/<包名>/files/carlink_state.txt`：断开时 `0\|-1`，连着时 `1\|0`/`1\|1`；
   车机切深浅色后 **5 秒内**（native `TICK_MS=5000`）第二个值变，JS 侧再 2 秒读到。
3. 车机歌词：播放页开关打开 → 车机端逐句跟随；关掉蓝牙歌词不应把车机歌词一起停（这是 A 层那个
   `IsShowBluetoothLyric.tsx` 联动修复的意义）。
4. 主题跟随：车机切深色 → 手机主题在约 7 秒内变；拔线 → 回浅色并停止请求定位（`sunLocation` 只在连着时发请求）。

---

## §B6 升级到更新版本的 checklist

按顺序做，每步没做完不要跳下一步。

1. **先只做 A 层**：取新版官方源码树，把本次 22 处改动逐条搬过去（有 git 的话 `git cherry-pick` 最省事），
   冲突按锚点串手工解决。锚点是「上下文唯一」式的，重点四条：
   `'common.isAutoTheme': false,`、`'player.isShowBluetoothFullLyric'` 附近、
   `<IsAutoTheme />`、`<IsShowBluetoothLyric />` 挂载点、`NowPlayingTitles` 里的 `lyric?: string`。
2. **跑一遍闸门**：`patch191_*_js.py` 若报错就是锚点不唯一 —— 那是好事，说明上游动了这些地方，
   手工看 diff，别放宽「命中次数必须为 1」的断言。
3. **重编 bundle**（§B3.2），并跑 `check_lang_params.py` / `measure_text_width.py`。
4. **重新定位 C 层 8 处锚点**：`l4/d`、`k4/b`、`s4/d` 这些混淆名**大概率会变**，
   正确认法是拿新 APK 的 `classes.dex` 搜特征串（本次官方 1.9.2 的实测特征：
   `lyricLine` 不存在、`android.media.metadata.LYRICS` 存在、`ucar.media.metadata` 不存在），
   再用 `native/smali/diff/` 的三个 diff 作模板逐处对位。4+3+1 的挂钩形状不变，变的只是类名/方法名/寄存器号。
5. **核对 `method_ids` 余量**：新版若已顶到 65536 上限，注入的 +1 就放不下 —— 那时要把注入逻辑整体挪进
   `classes2.dex`，或换 donor（如拆 ABI）。这是「能不能平移」的真正闸门，不是混淆名。
6. **重编 `classes2.dex`**（`native/java` 那份不用改，除非上游换 provider authority 规则）；
   桩类 `ContentObserver.java` 的方法签名字面量若与新版框架类不一致会 `NoSuchMethodError`。
7. **重打包 + 全量对答案**：§B5三张表逐条走一遍；`versionCode`/`versionName` 按新 donor 重新读，
   别抄 79003。
8. **实机验收**：§B5.4 四条，尤其第 2、3 条（协议键与开关联动最容易在新版本上悄悄失效）。

---

## §B7 故障速查

| 症状 | 原因 | 处置 |
| --- | --- | --- |
| 补丁打出 `23 files changed, 789 insertions` | 套到的是含 `constant.ts` 那行改动的那两份补丁 | 用随源码包交付的原包名补丁：22 文件 / +788 / −8 |
| `git apply` / `git apply -R` 报 does not apply | 树不匹配（补丁包名与树不一致，或基线不是 `v1.9.2`） | `git apply --check` 定位；必要时 `git checkout v1.9.2 -- src` |
| 走原包名时 `[8] 校验` 报 `bundle 里还留着旧东西: cn.toside.music.mobile.provider` | 禁串表是为换包名那条线写的，与原包名的必含串对撞 | 按 §B3.1 摘掉/条件化那一条（`rebuild191.py:501`、`bundle191.py:168`），别放宽整张表 |
| 跑完 `bundle191.py` 后 `git status` 一片红 | 它不加 `--keep` 时会用上游源码 zip 覆盖 `<tree>/src` | 在专门构建树里跑，或跑完 `git checkout -- src` 复原（见 §B3.2 B2） |
| 在已打补丁的源码树上重跑 `patch191_suntheme_js.py` 报「重复插入」 | 驱动是「锚点唯一 + 命中一次」式，本就是防重入的保护 | 源码树不用再跑驱动，直接 Metro + hermesc（§B3.2 B1） |
| `no main manifest attribute` | 用了 `java -jar baksmali.jar` | 按脚本写法：`java -Xmx3g -cp tools/baksmali.jar:tools/libs/* org.jf.baksmali.Main d <dex> -o <dir>` |
| `method_ids` 读出 610792 之类离谱数 | 读的是偏移 `0x58`（size），数量在 `0x5c` | 改读 `0x5c`（本文 §A9 已记这一条） |
| 启动即闪退 `NoClassDefFoundError` | 删了 `classes2.dex` 但 `classes.dex` 的挂钩还在 | 两层一起还原（§B4.4） |
| 装不上，报签名冲突（`INSTALL_FAILED_UPDATE_INCOMPATIBLE`） | 换了 keystore，与已装版本不同签 | 继续用 `scripts/android-debug.keystore`；或先卸载（数据会丢） |
| 车机歌词不显示、但没报错 | bundle 是官方版（`lyricLine` 从没被写入），或 `player.isShowCarWithLyric` 没打开 | 按 §B5.3 验 bundle 必含串；确认开关为真 |
| 关掉蓝牙歌词后车机歌词也停了 | 用了没带联动修复的旧 bundle | 还原成源码树里那份 `IsShowBluetoothLyric.tsx` 并重编 |
| 主题不跟深浅色切、也不报错 | `ucar_night_mode` 读到 `-1`（未装 CarWith / 键名随车机版本变） | `settings get global ucar_night_mode` 实测；键名来自 com.miui.carlink 4.0.20，换版本要重新抓 |
| `carlink_state.txt` 永远 `0\|-1` | native 轮询线程死了（某次异常逃出循环）或 `classes2.dex` 没装进去 | 验 §B5.2 的 `classes2.dex` 类数；`adb shell run-as`/`logcat` 找 `CarLinkBridge` 线程 |
| 成品里出现 `:\Users\` 或本机路径 | hermesc 被以绝对路径调用 | `cd` 到输出目录用相对文件名（§B3.2） |
| sha256 与交付件不符 | smali 重汇编排列不确定，正常 | 按 §B5.1-§B5.3 验**内容与改动面**，不验哈希 |

---

## §B8 校验值与可抄文本

### §B8.1 补丁与交付件

| 文件 | sha256 |
| --- | --- |
| `lx-music-1.9.2-CarWith-src-原包名-cn.toside.music.mobile.patch`（随源码包交付） | `26067febb82ce7ee14455b7d115d6b5cc2b403bed6724e7c404f7199012dfde7` |
| 源码树内 758 个文件 + 这片补丁 | 见源码包根目录 `SHA256SUMS.txt`（759 条） |
| `classes2.dex`（遗留成品 APK 内同一份） | `727c0c2252d13cece3ec3df33baaf0a8fbe7b550506d2ea02288df56148c72c1`（md5 `1972dab8bb7822e6b278850175c039f1`）。工具包的 `prebuilt/` 里有这一份；《APK 哈希参考》只列两个 APK |
| 签名证书 | `fac61745dc0903786fb9ede62a962b399f7348f0bb6f899b8332667591033b9c` |

工具包 `docs/APK哈希参考.txt` 里那两个 APK 哈希（`c2eb4fec…` / `8f0f15ff…`）属于遗留的那两份成品，
构建机产物目录还有一对同名件（`b4bdbe89…` / `c60600e8…`），字节不同（只差 `classes.dex` 的排列及其派生的
`META-INF/MANIFEST.MF`/`.SF`/`.RSA`），smali 侧差异 0、dex header 计数相同 —— 同一份代码、不同排列。
**原包名的 APK 没有哈希可记，因为它还没被构建过**；你构建出来后，用 §B5 那张表验内容与改动面，别找参照哈希。

### §B8.2 三个设置键（新增，默认全 false）

```
player.isShowCarWithLyric   车机逐句歌词开关（播放设置页）
common.followCarWithTheme   连着 CarWith 时按车机上报深浅色切主题（主题设置页）
common.followSunTheme       连着 CarWith 时按所在城市日出日落切主题（主题设置页）
```

`version: '2.1'` 沿用上游，三个键都带默认值，旧配置升级不需要迁移逻辑；
卸载重装后回到默认（全关），与官方行为一致。

### §B8.3 A 层改动的说明文字（写 commit message 或 PR 描述时可抄）

```
feat(carwith): 接入 CarWith/Jovi InCar 车机歌词与主题跟随（JS/TS 层）

基于官方 v1.9.2（tag v1.9.2 = commit 0669d1b6）的纯 JS/TS 改动，22 个文件 +788/-8。

新增设置项 3 个：
- player.isShowCarWithLyric  是否把歌词交给车机端逐句显示
- common.followCarWithTheme  连着 CarWith 时按车机上报的深色/浅色切主题
- common.followSunTheme      连着 CarWith 时按所在城市日出日落切主题

歌词通道：core/init/player/lyric.ts 走 updateNowPlayingTitles({ ..., lyricLine })，
utils.ts 的 NowPlayingTitles 增加 lyricLine 字段，经 NativePlayer 的
setPlaybackMetadata 传给 native，最终由 classes2.dex 写入
MediaMetadataCompat 的 ucar.media.metadata.LYRICS_LINE / LYRICS_WHOLE。

主题跟随：core/init/sunTheme.ts + utils/sunState.ts / sunLocation.ts / sunrise.ts
轮询 ExternalDirectoryPath/carlink_state.txt（native 侧 5s 写、JS 侧 2s 读），
得到"是否连接 CarWith + 车机深浅色"，据此调用 theme 切换。

本提交只含源码层改动；dex/smali 注入、重打包与签名不在这里，
native 层源码与施工脚本不在源码树里，见本文附录 A/C 与工具包。
```

---

## §B9 交接给下一个工程师 / AI 的三句话

1. **A 层看 git，B/C/D 层看 APK 条目**：git 历史里没有字节码，别拿 `git diff` 的结论去断言手机上跑的是什么。
2. **别信哈希**：`classes.dex` 不可字节复现，验收一律走§B5的改动面 + 内容断言，最后 §B5.4 实机。
3. **版本号一换，先重找锚点**：混淆名不可平移；`method_ids` 余量才是能不能继续注入的硬闸门。


---

# 附录

以下三份是「脱离工具包也能重建 native 层」的最小材料：桥接类全文、编译期桩类全文、
以及 `classes.dex` 上三处注入的逐字 diff（对 baksmali 2.5.2 无调试信息反汇编出的
`l4/d.smali` / `k4/b.smali` / `MainApplication.smali` 生成，换版本时按 §B6 重新对位）。

## 附录 A　`CarLinkBridge.java`（classes2.dex 的唯一真源码，282 行）

```java
package com.salt.music.bridge;

import android.database.ContentObserver;
import java.io.ByteArrayOutputStream;
import java.io.File;
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.OutputStream;
import java.io.OutputStreamWriter;
import java.io.Writer;
import java.lang.reflect.Array;
import java.lang.reflect.Method;

/**
 * 把「手机现在有没有通过 CarWith 连着车机」和「车机自己是浅色还是深色」写成文件给 JS 读。
 *
 * 连接判据是 CarWith 连上车机时会在手机上建出的虚拟屏（名字是 com.miui.carlink /
 * com.xiaomi.ucar.minimap）。这个判据实测可靠：连上就出现、断开 2 秒内消失、重连换 id。
 * 走这条路是为了不申请任何新权限、不改清单（蓝牙/网络状态都要动清单）。
 *
 * 车机深浅色读 CarWith 自己写进 Settings.Global 的 `ucar_night_mode`（0=浅色、1=深色）：
 * 那个键本来就是它写给百度 CarLife 跨应用读的，第三方读 Settings.Global 不需要权限。
 * 只在连着车机时去读，断开一个查询都不发；读不到写 -1，由 JS 决定退回哪套判断。
 * 给它挂 ContentObserver，车机一写就立刻落到文件里，不用等满一轮。
 *
 * 文件内容一行两个值：`连接(0/1)|深浅(-1|0|1)`。
 * 除 ContentObserver（编译桩在 jsrc191/android/database 下，运行时是系统真类）外全部走反射，
 * 所以不需要 android.jar 就能编译。任何异常都只让桥接自己停掉，绝不影响洛雪本体。
 */
public final class CarLinkBridge {

    /** CarWith 写深浅色用的 Settings.Global 键（从 com.miui.carlink 4.0.20 里读出来的字面量） */
    private static final String NIGHT_KEY = "ucar_night_mode";

    private static final long TICK_MS = 5000L;

    private static volatile boolean started = false;

    // 车机切深浅色时由观察者把轮询叫醒，不用等满 5 秒
    private static final Object WAKE = new Object();
    private static volatile boolean sWake = false;
    private static volatile boolean sObserving = false;
    /** 观察者必须留强引用，被 GC 掉系统就不通知了 */
    private static ContentObserver sObserver;

    private CarLinkBridge() {
    }

    public static void start(Object appContext) {
        if (started) {
            return;
        }
        started = true;
        final Object ctx = appContext;
        Thread t = new Thread(new Runnable() {
            public void run() {
                loop(ctx);
            }
        }, "CarLinkBridge");
        t.setDaemon(true);
        t.start();
    }

    /** 观察者只做一件事：叫醒轮询线程。读写文件都留在那边做，主线程不碰磁盘 */
    private static final class NightObserver extends ContentObserver {

        NightObserver() {
            // threadAware=true：回调在主线程派发
            super(true);
        }

        public void onChange(boolean selfChange) {
            synchronized (WAKE) {
                sWake = true;
                WAKE.notifyAll();
            }
        }
    }

    /**
     * 线程是守护线程，随进程结束；这里**任何一次失败都不能退出循环**——
     * 线程一死，carlink_state.txt 就永远停在最后一次写下的值上，
     * 「跟随车机深浅色」和「连接时跟随日出日落」都会静默失效（连不上也不断开）。
     * 所以启动太早拿不到 DisplayManager、或写盘临时失败，都留到下一轮再试。
     */
    private static void loop(Object ctx) {
        Object dm = null;
        File file = null;
        boolean primed = false;
        while (true) {
            try {
                if (file == null) {
                    Object dir = call(ctx, "getExternalFilesDir", new Class[] { String.class },
                            new Object[] { null });
                    if (dir != null) {
                        file = new File((File) dir, "carlink_state.txt");
                    }
                }
                if (dm == null) {
                    dm = call(ctx, "getSystemService", new Class[] { String.class },
                            new Object[] { "display" });
                }
                if (file != null && !primed) {
                    // 开机第一件事先把上次的残留按「没连着」盖掉：这个文件跨进程重启留存，
                    // 车上断电或被系统杀掉时它可能还写着 `1|1`。JS 那侧 2 秒一轮，抢在本线程第一轮之前
                    // 读到旧值就会以为还连着车机，于是发出 IP 请求、并把主题钉在旧的车机深浅色上。
                    // DisplayManager 要等，目录不用等，所以这里不等它就先写一次。
                    primed = true;
                    write(file, "0|-1");
                }
                if (file != null && dm != null) {
                    if (!sObserving) {
                        // 挂上观察者后车机一切深浅色就能立刻反映到文件里；挂不上也不影响，下面还有轮询
                        observe(ctx);
                    }
                    boolean up = isCarWithUp(dm);
                    // 没连着车机就不去查 Settings，一个查询都不发
                    String now = up ? "1|" + nightMode(ctx) : "0|-1";
                    // 只在值真的变了、或文件被清掉/读不出的时候才写盘：
                    // 一直连着且车机没切深浅色、或一直不连着时，一次写都不做
                    if (!now.equals(read(file))) {
                        write(file, now);
                    }
                }
            } catch (Throwable ignored) {
            }
            sleep(TICK_MS);
        }
    }

    /**
     * 给 `ucar_night_mode` 挂 ContentObserver。挂在桥接线程上做，失败就算了——
     * 5 秒一轮的轮询本来就是兜底。
     */
    private static void observe(Object ctx) {
        try {
            Object resolver = call(ctx, "getContentResolver", null, null);
            if (resolver == null) {
                return;
            }
            Class<?> global = Class.forName("android.provider.Settings$Global");
            Object uri = global.getMethod("getUriFor", String.class).invoke(null, NIGHT_KEY);
            if (uri == null) {
                return;
            }
            NightObserver observer = new NightObserver();
            resolver.getClass().getMethod("registerContentObserver",
                    Class.forName("android.net.Uri"), boolean.class, ContentObserver.class)
                    .invoke(resolver, new Object[] { uri, Boolean.TRUE, observer });
            sObserver = observer;
            sObserving = true;
        } catch (Throwable ignored) {
            // 挂不上（ROM 没有这个 API、resolver 还没好）就下一轮再试，行为退回纯轮询
        }
    }

    /**
     * 车机当前的深浅色：0=浅色、1=深色；读不到（没装 CarWith、键还不存在、查询失败）返回 -1，
     * 由 JS 退回按日出日落判断。
     */
    private static int nightMode(Object ctx) {
        try {
            Object resolver = call(ctx, "getContentResolver", null, null);
            if (resolver == null) {
                return -1;
            }
            Class<?> global = Class.forName("android.provider.Settings$Global");
            Method m = global.getMethod("getInt", Class.forName("android.content.ContentResolver"),
                    String.class, int.class);
            Object v = m.invoke(null, resolver, NIGHT_KEY, Integer.valueOf(-1));
            return v == null ? -1 : ((Integer) v).intValue();
        } catch (Throwable t) {
            return -1;
        }
    }

    private static boolean isCarWithUp(Object dm) {
        try {
            Object arr = call(dm, "getDisplays", null, null);
            if (arr == null) {
                return false;
            }
            int n = Array.getLength(arr);
            for (int i = 0; i < n; i++) {
                Object d = Array.get(arr, i);
                // 必须真是虚拟屏（type==5=DISPLAY_TYPE_VIRTUAL），只看名字会把别的屏误判成已连接
                Object type = call(d, "getType", null, null);
                if (type == null || ((Integer) type).intValue() != 5) {
                    continue;
                }
                Object nameObj = call(d, "getName", null, null);
                if (nameObj == null) {
                    continue;
                }
                String name = nameObj.toString().toLowerCase();
                if (name.contains("carlink") || name.contains("ucar")) {
                    return true;
                }
            }
        } catch (Throwable t) {
            return false;
        }
        return false;
    }

    /** 读不出内容（文件不存在、被清掉、读到一半被改）一律返回 null，调用方当成「需要重写」 */
    private static String read(File file) {
        FileInputStream in = null;
        try {
            in = new FileInputStream(file);
            ByteArrayOutputStream buf = new ByteArrayOutputStream();
            byte[] chunk = new byte[64];
            int n;
            while ((n = in.read(chunk)) > 0) {
                buf.write(chunk, 0, n);
            }
            return new String(buf.toByteArray(), "US-ASCII").trim();
        } catch (Throwable t) {
            return null;
        } finally {
            try {
                if (in != null) {
                    in.close();
                }
            } catch (Throwable ignored) {
            }
        }
    }

    /**
     * 先写同名 .tmp 再 rename：直接截断重写的话，JS 正好撞上这一刻会读到空串，
     * 那个周期就被当成「没连车机」，主题白跳一次。rename 在同一目录里是原子的。
     */
    private static void write(File file, String text) throws Exception {
        File tmp = new File(file.getParentFile(), file.getName() + ".tmp");
        Writer w = new OutputStreamWriter(new FileOutputStream(tmp), "US-ASCII");
        try {
            w.write(text);
            w.flush();
        } finally {
            w.close();
        }
        if (tmp.renameTo(file)) {
            return;
        }
        // 换名失败（比如目录不允许 rename）就退回直接写，不能因此把状态丢掉
        tmp.delete();
        OutputStream direct = new FileOutputStream(file);
        try {
            direct.write(text.getBytes("US-ASCII"));
            direct.flush();
        } finally {
            direct.close();
        }
    }

    private static Object call(Object target, String name, Class[] types, Object[] args) {
        if (target == null) {
            return null;
        }
        try {
            Method m = types == null ? target.getClass().getMethod(name)
                    : target.getClass().getMethod(name, types);
            return m.invoke(target, args);
        } catch (Throwable t) {
            return null;
        }
    }

    /** 睡着时被观察者叫醒就立刻返回，不用等满一轮；没人叫醒就照常睡 ms 毫秒 */
    private static void sleep(long ms) {
        synchronized (WAKE) {
            if (!sWake) {
                try {
                    WAKE.wait(ms);
                } catch (InterruptedException e) {
                }
            }
            sWake = false;
        }
    }
}
```

## 附录 B　`ContentObserver.java`（编译期桩类，16 行，绝不能进 dex）

```java
package android.database;

/**
 * 编译桩：运行时用的是系统里那个真类。
 * 只要声明我们用到的构造方法与回调，签名的字面量描述符必须和真类逐字符一致，
 * 否则运行时就是 NoSuchMethodError。threadAware=true 表示回调在主线程执行。
 * 这个目录下的东西只参与编译，不能进 dex（框架类由系统提供，自己带一份会撞名）。
 */
public abstract class ContentObserver {

  public ContentObserver(boolean threadAware) {
  }

  public void onChange(boolean selfChange) {
  }
}
```

## 附录 C　`classes.dex` 三处注入的逐字 diff

### C.1 `l4/d.smali`（TrackMetadata：读 `lyricLine`、写 `ucar.media.metadata.LYRICS_LINE` / `LYRICS_WHOLE`）

```diff
--- a/l4/d.smali (官方 1.9.2)
+++ b/l4/d.smali (成品)
@@ -22,6 +22,8 @@
 
 .field public i:Landroid/support/v4/media/RatingCompat;
 
+.field public j:Ljava/lang/String;
+
 
 # direct methods
 .method public constructor <init>()V
@@ -154,7 +156,7 @@
 
     move-result v0
 
-    if-eqz v0, :cond_76
+    if-eqz v0, :cond_7e
 
     invoke-virtual {p2, p1}, Landroid/os/BaseBundle;->getString(Ljava/lang/String;)Ljava/lang/String;
 
@@ -162,14 +164,22 @@
 
     iput-object p1, p0, Ll4/d;->h:Ljava/lang/String;
 
-    :cond_76
+    const-string p1, "lyricLine"
+
+    invoke-virtual {p2, p1}, Landroid/os/BaseBundle;->getString(Ljava/lang/String;)Ljava/lang/String;
+
+    move-result-object p1
+
+    iput-object p1, p0, Ll4/d;->j:Ljava/lang/String;
+
+    :cond_7e
     const-string p1, "rating"
 
     invoke-virtual {p2, p1}, Landroid/os/BaseBundle;->containsKey(Ljava/lang/String;)Z
 
     move-result v0
 
-    if-eqz v0, :cond_84
+    if-eqz v0, :cond_8c
 
     invoke-static {p2, p1, p3}, Lj4/h;->c(Landroid/os/Bundle;Ljava/lang/String;I)Landroid/support/v4/media/RatingCompat;
 
@@ -177,7 +187,7 @@
 
     iput-object p1, p0, Ll4/d;->i:Landroid/support/v4/media/RatingCompat;
 
-    :cond_84
+    :cond_8c
     return-void
 .end method
 
@@ -227,22 +237,39 @@
     invoke-virtual {v0, v2, v1}, Landroid/support/v4/media/MediaMetadataCompat$b;->e(Ljava/lang/String;Ljava/lang/String;)Landroid/support/v4/media/MediaMetadataCompat$b;
 
     :cond_31
+    iget-object v1, p0, Ll4/d;->j:Ljava/lang/String;
+
+    if-eqz v1, :cond_43
+
+    const-string v2, "ucar.media.metadata.LYRICS_LINE"
+
+    invoke-virtual {v0, v2, v1}, Landroid/support/v4/media/MediaMetadataCompat$b;->e(Ljava/lang/String;Ljava/lang/String;)Landroid/support/v4/media/MediaMetadataCompat$b;
+
+    iget-object v1, p0, Ll4/d;->h:Ljava/lang/String;
+
+    if-eqz v1, :cond_43
+
+    const-string v2, "ucar.media.metadata.LYRICS_WHOLE"
+
+    invoke-virtual {v0, v2, v1}, Landroid/support/v4/media/MediaMetadataCompat$b;->e(Ljava/lang/String;Ljava/lang/String;)Landroid/support/v4/media/MediaMetadataCompat$b;
+
+    :cond_43
     iget-wide v1, p0, Ll4/d;->g:J
 
     const-wide/16 v3, 0x0
 
     cmp-long v5, v1, v3
 
-    if-lez v5, :cond_3e
+    if-lez v5, :cond_50
 
     const-string v3, "android.media.metadata.DURATION"
 
     invoke-virtual {v0, v3, v1, v2}, Landroid/support/v4/media/MediaMetadataCompat$b;->c(Ljava/lang/String;J)Landroid/support/v4/media/MediaMetadataCompat$b;
 
-    :cond_3e
+    :cond_50
     iget-object v1, p0, Ll4/d;->a:Landroid/net/Uri;
 
-    if-eqz v1, :cond_4b
+    if-eqz v1, :cond_5d
 
     const-string v2, "android.media.metadata.ART_URI"
 
@@ -252,15 +279,15 @@
 
     invoke-virtual {v0, v2, v1}, Landroid/support/v4/media/MediaMetadataCompat$b;->e(Ljava/lang/String;Ljava/lang/String;)Landroid/support/v4/media/MediaMetadataCompat$b;
 
-    :cond_4b
+    :cond_5d
     iget-object v1, p0, Ll4/d;->i:Landroid/support/v4/media/RatingCompat;
 
-    if-eqz v1, :cond_54
+    if-eqz v1, :cond_66
 
     const-string v2, "android.media.metadata.RATING"
 
     invoke-virtual {v0, v2, v1}, Landroid/support/v4/media/MediaMetadataCompat$b;->d(Ljava/lang/String;Landroid/support/v4/media/RatingCompat;)Landroid/support/v4/media/MediaMetadataCompat$b;
 
-    :cond_54
+    :cond_66
     return-object v0
 .end method
```

### C.2 `k4/b.smali`（MetadataManager：`ucZ` 标记 + 断开时清键，`.registers` 5→6）

```diff
--- a/k4/b.smali (官方 1.9.2)
+++ b/k4/b.smali (成品)
@@ -3,6 +3,10 @@
 .source "SourceFile"
 
 
+# static fields
+.field static ucZ:Z
+
+
 # instance fields
 .field private final a:Lcom/guichaguri/trackplayer/service/MusicService;
 
@@ -1157,7 +1161,7 @@
 .end method
 
 .method public v(Landroid/os/Bundle;)V
-    .registers 5
+    .registers 6
 
     iget-object v0, p0, Lk4/b;->k:Landroid/support/v4/media/MediaMetadataCompat$b;
 
@@ -1166,13 +1170,73 @@
     return-void
 
     :cond_5
+    const-string v3, "lyricLine"
+
+    invoke-virtual {p1, v3}, Landroid/os/BaseBundle;->getString(Ljava/lang/String;)Ljava/lang/String;
+
+    move-result-object v0
+
+    if-eqz v0, :cond_27
+
+    const-string v3, "lyric"
+
+    invoke-virtual {p1, v3}, Landroid/os/BaseBundle;->getString(Ljava/lang/String;)Ljava/lang/String;
+
+    move-result-object v1
+
+    if-eqz v1, :cond_1c
+
+    const-string v3, "ucar.media.metadata.LYRICS_WHOLE"
+
+    iget-object v2, p0, Lk4/b;->k:Landroid/support/v4/media/MediaMetadataCompat$b;
+
+    invoke-virtual {v2, v3, v1}, Landroid/support/v4/media/MediaMetadataCompat$b;->e(Ljava/lang/String;Ljava/lang/String;)Landroid/support/v4/media/MediaMetadataCompat$b;
+
+    :cond_1c
+    const-string v3, "ucar.media.metadata.LYRICS_LINE"
+
+    iget-object v2, p0, Lk4/b;->k:Landroid/support/v4/media/MediaMetadataCompat$b;
+
+    invoke-virtual {v2, v3, v0}, Landroid/support/v4/media/MediaMetadataCompat$b;->e(Ljava/lang/String;Ljava/lang/String;)Landroid/support/v4/media/MediaMetadataCompat$b;
+
+    const/4 v1, 0x1
+
+    sput-boolean v1, Lk4/b;->ucZ:Z
+
+    goto :goto_3e
+
+    :cond_27
+    sget-boolean v1, Lk4/b;->ucZ:Z
+
+    if-eqz v1, :cond_3e
+
+    const/4 v1, 0x0
+
+    sput-boolean v1, Lk4/b;->ucZ:Z
+
+    const-string v0, ""
+
+    const-string v3, "ucar.media.metadata.LYRICS_WHOLE"
+
+    iget-object v2, p0, Lk4/b;->k:Landroid/support/v4/media/MediaMetadataCompat$b;
+
+    invoke-virtual {v2, v3, v0}, Landroid/support/v4/media/MediaMetadataCompat$b;->e(Ljava/lang/String;Ljava/lang/String;)Landroid/support/v4/media/MediaMetadataCompat$b;
+
+    const-string v3, "ucar.media.metadata.LYRICS_LINE"
+
+    iget-object v2, p0, Lk4/b;->k:Landroid/support/v4/media/MediaMetadataCompat$b;
+
+    invoke-virtual {v2, v3, v0}, Landroid/support/v4/media/MediaMetadataCompat$b;->e(Ljava/lang/String;Ljava/lang/String;)Landroid/support/v4/media/MediaMetadataCompat$b;
+
+    :cond_3e
+    :goto_3e
     const-string v0, "title"
 
     invoke-virtual {p1, v0}, Landroid/os/BaseBundle;->containsKey(Ljava/lang/String;)Z
 
     move-result v1
 
-    if-eqz v1, :cond_18
+    if-eqz v1, :cond_51
 
     iget-object v1, p0, Lk4/b;->k:Landroid/support/v4/media/MediaMetadataCompat$b;
 
@@ -1184,14 +1248,14 @@
 
     invoke-virtual {v1, v2, v0}, Landroid/support/v4/media/MediaMetadataCompat$b;->e(Ljava/lang/String;Ljava/lang/String;)Landroid/support/v4/media/MediaMetadataCompat$b;
 
-    :cond_18
+    :cond_51
     const-string v0, "artist"
 
     invoke-virtual {p1, v0}, Landroid/os/BaseBundle;->containsKey(Ljava/lang/String;)Z
 
     move-result v1
 
-    if-eqz v1, :cond_2b
+    if-eqz v1, :cond_64
 
     iget-object v1, p0, Lk4/b;->k:Landroid/support/v4/media/MediaMetadataCompat$b;
 
@@ -1203,14 +1267,14 @@
 
     invoke-virtual {v1, v2, v0}, Landroid/support/v4/media/MediaMetadataCompat$b;->e(Ljava/lang/String;Ljava/lang/String;)Landroid/support/v4/media/MediaMetadataCompat$b;
 
-    :cond_2b
+    :cond_64
     const-string v0, "album"
 
     invoke-virtual {p1, v0}, Landroid/os/BaseBundle;->containsKey(Ljava/lang/String;)Z
 
     move-result v1
 
-    if-eqz v1, :cond_3e
+    if-eqz v1, :cond_77
 
     iget-object v1, p0, Lk4/b;->k:Landroid/support/v4/media/MediaMetadataCompat$b;
 
@@ -1222,14 +1286,14 @@
 
     invoke-virtual {v1, v2, v0}, Landroid/support/v4/media/MediaMetadataCompat$b;->e(Ljava/lang/String;Ljava/lang/String;)Landroid/support/v4/media/MediaMetadataCompat$b;
 
-    :cond_3e
+    :cond_77
     const-string v0, "lyric"
 
     invoke-virtual {p1, v0}, Landroid/os/BaseBundle;->containsKey(Ljava/lang/String;)Z
 
     move-result v1
 
-    if-eqz v1, :cond_51
+    if-eqz v1, :cond_8a
 
     iget-object v1, p0, Lk4/b;->k:Landroid/support/v4/media/MediaMetadataCompat$b;
 
@@ -1241,7 +1305,7 @@
 
     invoke-virtual {v1, v2, p1}, Landroid/support/v4/media/MediaMetadataCompat$b;->e(Ljava/lang/String;Ljava/lang/String;)Landroid/support/v4/media/MediaMetadataCompat$b;
 
-    :cond_51
+    :cond_8a
     iget-object p1, p0, Lk4/b;->c:Landroid/support/v4/media/session/MediaSessionCompat;
 
     iget-object v0, p0, Lk4/b;->k:Landroid/support/v4/media/MediaMetadataCompat$b;
```

### C.3 `MainApplication.smali`（onCreate 里一行挂钩）

```diff
--- a/cn/toside/music/mobile/MainApplication.smali (官方 1.9.2)
+++ b/cn/toside/music/mobile/MainApplication.smali (成品)
@@ -37,6 +37,8 @@
 
     invoke-super {p0}, Ls4/d;->onCreate()V
 
+    invoke-static {p0}, Lcom/salt/music/bridge/CarLinkBridge;->start(Ljava/lang/Object;)V
+
     invoke-virtual {p0}, Lcn/toside/music/mobile/MainApplication;->a()Lcom/facebook/react/C;
 
     move-result-object v0
```

## 附录 D　新增的 7 个源码文件

| 路径 | 行数 | 作用 |
| --- | --- | --- |
| `src/utils/sunState.ts` | 10 | 模块级可变状态 `{enabled, night}`，在主题计算与车机轮询之间传值；刻意不走 redux，避免每 2 秒轮询触发一次全局重渲染 |
| `src/utils/sunrise.ts` | 90 | 内置日出日落算法，按经纬度 + 日期现算升落时刻；极圈算不出时上层显示「极圈地区」 |
| `src/utils/sunLocation.ts` | 294 | IP 定位 + 调度策略 + 状态文案；只在连着 CarWith 期间发请求，缓存 key `@salt_sun_location` / `@salt_sun_budget` |
| `src/core/init/sunTheme.ts` | 120 | 每 2 秒读一次 `carlink_state.txt`，维护 sunState 并驱动主题切换与状态行 |
| `src/screens/Home/Views/Setting/settings/Theme/FollowCarWithTheme.tsx` | 34 | 主题设置页「跟随CarWith亮、暗模式切换主题」开关行 |
| `src/screens/Home/Views/Setting/settings/Theme/FollowSunTheme.tsx` | 101 | 「CarWith连接时跟随日出日落切换主题」开关行 + 下面那行五态状态文字 |
| `src/screens/Home/Views/Setting/settings/Player/IsShowCarWithLyric.tsx` | 48 | 播放设置页的车机歌词开关；勾上时 `showRemoteLyric(true)`，失败把开关落回 false |

这 7 个文件在 git 里是**新增**（`git apply -R` 撤销时会把内容还原成删除，但空目录与文件本身要手工 `rm` 才干净，见 §B4.1）。

## 附录 E　三个设置项与两条车机协议

| 类型 | 键 / 字段 | 取值与含义 |
| --- | --- | --- |
| 设置项（本改动新增） | `player.isShowCarWithLyric` | 默认 `false`；真则把当前句歌词交给车机逐句显示 |
| 设置项（本改动新增） | `common.followCarWithTheme` | 默认 `false`；连着 CarWith 时按车机上报的深浅色切主题 |
| 设置项（本改动新增） | `common.followSunTheme` | 默认 `false`；连着 CarWith 时按所在城市日出日落切主题 |
| `Settings.Global` | `ucar_night_mode` | `0` 浅色 / `1` 深色 / `-1` 读不到（未装 CarWith、键不存在、查询失败）；键名取自 com.miui.carlink 4.0.20 的字面量 |
| `MediaMetadataCompat` 扩展 | `ucar.media.metadata.LYRICS_LINE` | 当前这一句 |
| `MediaMetadataCompat` 扩展 | `ucar.media.metadata.LYRICS_WHOLE` | 整首歌词（车机可提前排上下句） |
| 跨进程状态文件 | `carlink_state.txt`（RNFS `ExternalDirectoryPath`，即 `/sdcard/Android/data/<包名>/files`） | 一行两值 `连接(0/1)|深浅(-1|0|1)`；native 每 5 秒写、变化才写，JS 每 2 秒读 |

车机连接判定：遍历 `DisplayManager.getDisplays()`，**必须 `getType()==5`（DISPLAY_TYPE_VIRTUAL）
且 `getName().toLowerCase()` 含 `carlink` 或 `ucar`** —— 只看名字会误判（投屏、HDMI 也是 virtual）。
