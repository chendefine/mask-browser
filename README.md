# mask-browser — Fingerprint Chromium for Windows

基于 **ungoogled-chromium 整套构建体系** 打包的 [fingerprint-chromium](https://github.com/chendefine/fingerprint-chromium)
**148.0.7778.215** 指纹浏览器,目标平台 **Windows x64**。

本仓库不包含 Chromium 源码,也不做本地构建 —— 全部构建通过本仓库的
**GitHub Actions CI** 完成(Windows runner 上原生构建,分多个 stage 续跑,绕过单 job 6 小时限制)。

## 组成

| 部分 | 来源 | 说明 |
|---|---|---|
| 构建体系 | [ungoogled-chromium-windows](https://github.com/ungoogled-software/ungoogled-chromium-windows) @ `148.0.7778.215-1.1` | Windows 平台的 ungoogled-chromium 官方移植(剪枝、域名替换、Windows 专属补丁、打包) |
| 补丁集 | submodule `ungoogled-chromium/` → [chendefine/fingerprint-chromium](https://github.com/chendefine/fingerprint-chromium) @ branch `148.0.7778.215` | ungoogled-chromium `148.0.7778.215-1` + `patches/extra/fingerprint/*` 指纹补丁(Canvas/WebGL/Audio/Font/UA/Timezone/WebDriver 等 18 个补丁) |

指纹补丁与 Windows 专属补丁修改的文件**零交集**,按 build.py 的顺序先应用
ungoogled-chromium(含指纹)补丁,再应用 Windows 补丁。

## 触发构建

```bash
gh workflow run build-x64.yml --ref main
```

流程(`.github/workflows/`):

1. `build-x64.yml` → 调用 `reusable-build.yml`(build-1 … build-16 链式 job,`windows-2022` runner)。
2. build-1:`clone.py` 拉取 Chromium 148.0.7778.215 源码与 DEPS、win64 PGO profiles →
   剪枝 → 依序应用补丁 → 域名替换 → 工具链(llvm/ninja/rust/esbuild 等,见 `downloads.ini`)→
   gn bootstrap → ninja 编译 `chrome` / `chromedriver` / `mini_installer`(3.5 小时超时)。
3. 超时中断时,stage action 把 `build/src` 打成 zip(保留时间戳,ninja 可增量续跑)上传为 artifact,
   下一个 stage job 下载后继续编译,直到完成(上游经验约 8–12 个 stage / 40–48 小时)。
4. 构建成功后 `package.py` 产出:
   - `fingerprint-chromium_148.0.7778.215-1.1_installer_x64.exe`(mini_installer 安装包)
   - `fingerprint-chromium_148.0.7778.215-1.1_windows_x64.zip`(便携版,含 chromedriver)
5. `release` job 下载产物并发布到 GitHub Release(`148.0.7778.215-fp1`)。

## 相对上游的改动

- `build.py`:ninja 超时中断写 `build/BUILD_INTERRUPTED` 标记;打包步骤改为 `check=True`。
- `.github/actions/stage/index.js`:node20;真实错误(非超时)立即置败,避免 16 个 stage 空转;
  产物名改为 `fingerprint-chromium*`。
- `package.py`:产物文件名前缀 `fingerprint-chromium_`。
- `.github/workflows/build-x64.yml`:新增 `release` job,构建完成后自动发布 Release。

## 本地目录(不参与 CI)

- `origin/fingerprint-chromium/`:指纹补丁源码副本(与 submodule 指向同一仓库同一提交)。
- `upstream-uc-win/`:上游 windows 移植仓库参考副本(tag `148.0.7778.215-1.1`)。
