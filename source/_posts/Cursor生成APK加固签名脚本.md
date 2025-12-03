---
title: Cursor生成APK加固签名脚本
date: 2025-12-03 10:40:57
tags: [android, 脚本]
cover: https://s2.loli.net/2025/12/03/QK5TxAPdvhgEBjF.jpg
categories: android
---

<meta name="referrer" content="no-referrer"/>

## 背景
> 为了提高APK的安全等级，要求APK使用高级加固方式进行加固。了解了市面上的主流加固方案发现都是收费的而且费用不低！最后选择了免费的OPPO加固。APK在加固后需要进行重新签名，签名流程分为好几个步骤并且涉及到配置android sdk、签名文件地址、apk地址等信息。为了提高工作效率需要生成一个脚本进行一键签名。

## 实践
分别用**DeepSeek、Trae**和**Cursor**进行生成脚本，最后进行比较发现**Cursor生成的最好**
生成脚本描述文本：
> **生成一个可以在win11电脑上运行的给加固后的apk进行签名的脚本**

以下是Cursor生成的信息：

***

这里给你一个可在 Windows 11 上直接使用的 PowerShell 签名脚本（支持加固后 APK）
将下面内容保存为 sign-apk.ps1，PowerShell 7 或 Windows PowerShell 都可运行。脚本会自动查找 Android SDK Build-Tools（或可手动指定），可选跳过 zipalign，支持选择签名方案 v1/v2/v3/v4。

```
param(
    [Parameter(Mandatory = $true)]
    [string]$ApkPath,                 # 待签名的APK路径

    [Parameter(Mandatory = $true)]
    [string]$KeystorePath,            # keystore 文件路径(.jks/.keystore)

    [Parameter(Mandatory = $true)]
    [string]$Alias,                   # key别名

    [Parameter(Mandatory = $true)]
    [string]$KeystorePass,            # keystore 密码

    [string]$KeyPass,                 # key 密码（如不填默认与 keystore 密码相同）

    [string]$BuildToolsDir,           # 指定 Android build-tools 目录（可选，如 C:\Android\sdk\build-tools\35.0.0）
    [string]$OutDir,                  # 输出目录（可选，默认与 APK 同目录）
    [switch]$SkipZipalign,            # 跳过 zipalign（部分加固包要求跳过时可用）
    [switch]$Overwrite,               # 输出存在时覆盖
    [ValidateSet("v1","v2","v3","v4","all")]
    [string]$Schemes = "all"          # 指定签名方案
)

Set-StrictMode -Version Latest
$ErrorActionPreference = "Stop"

function Write-Info($msg) { Write-Host "[INFO] $msg" -ForegroundColor Cyan }
function Write-Warn($msg) { Write-Host "[WARN] $msg" -ForegroundColor Yellow }
function Write-Err ($msg) { Write-Host "[ERROR] $msg" -ForegroundColor Red }

# 规范化路径
$ApkPath       = (Resolve-Path -LiteralPath $ApkPath).Path
$KeystorePath  = (Resolve-Path -LiteralPath $KeystorePath).Path
if ([string]::IsNullOrWhiteSpace($OutDir)) {
    $OutDir = Split-Path -Parent $ApkPath
} else {
    $OutDir = (Resolve-Path -LiteralPath $OutDir).Path
}
if ([string]::IsNullOrWhiteSpace($KeyPass)) { $KeyPass = $KeystorePass }

# 寻找 build-tools
function Get-LatestBuildToolsDir {
    $candidates = @()
    foreach ($envVar in @("ANDROID_HOME","ANDROID_SDK_ROOT")) {
        $sdk = [Environment]::GetEnvironmentVariable($envVar, "User")
        if (-not $sdk) { $sdk = [Environment]::GetEnvironmentVariable($envVar, "Machine") }
        if ($sdk) {
            $bt = Join-Path $sdk "build-tools"
            if (Test-Path $bt) { $candidates += $bt }
        }
    }
    foreach ($root in $candidates) {
        $subdirs = Get-ChildItem -Path $root -Directory -ErrorAction SilentlyContinue | Sort-Object Name -Descending
        if ($subdirs -and $subdirs.Count -gt 0) {
            return $subdirs[0].FullName
        }
    }
    return $null
}

if ([string]::IsNullOrWhiteSpace($BuildToolsDir)) {
    $BuildToolsDir = Get-LatestBuildToolsDir
    if (-not $BuildToolsDir) {
        Write-Err "未找到 Android build-tools，请安装 Android SDK 并设置 ANDROID_HOME/ANDROID_SDK_ROOT，或通过 -BuildToolsDir 指定。"
        exit 1
    }
}

# 定位工具（Windows 下 apksigner 为 .bat）
$zipalign   = Join-Path $BuildToolsDir "zipalign.exe"
$apksigner  = Join-Path $BuildToolsDir "apksigner.bat"
if (-not (Test-Path $zipalign)) { Write-Warn "未找到 zipalign：$zipalign，将无法对齐。建议安装完整 build-tools 或使用 -SkipZipalign。" }
if (-not (Test-Path $apksigner)) { Write-Err "未找到 apksigner：$apksigner"; exit 1 }

# 生成输出文件名
$baseName   = [IO.Path]::GetFileNameWithoutExtension($ApkPath)
$alignedApk = Join-Path $OutDir ($baseName + "_aligned.apk")
$signedApk  = Join-Path $OutDir ($baseName + "_signed.apk")

# 覆盖处理
if ($Overwrite) {
    foreach ($f in @($alignedApk,$signedApk)) {
        if (Test-Path $f) { Remove-Item -LiteralPath $f -Force -ErrorAction SilentlyContinue }
    }
} else {
    foreach ($f in @($alignedApk,$signedApk)) {
        if (Test-Path $f) {
            Write-Err "输出文件已存在：$f。使用 -Overwrite 覆盖或删除后重试。"
            exit 1
        }
    }
}

# 可选 zipalign
$inputForSign = $ApkPath
if (-not $SkipZipalign) {
    if (Test-Path $zipalign) {
        Write-Info "执行 zipalign..."
        & $zipalign -v -p 4 "$ApkPath" "$alignedApk" | Write-Host
        if ($LASTEXITCODE -ne 0 -or -not (Test-Path $alignedApk)) {
            Write-Err "zipalign 失败。可尝试使用 -SkipZipalign 跳过或检查 APK。"
            exit 1
        }
        $inputForSign = $alignedApk
    } else {
        Write-Warn "找不到 zipalign，跳过对齐。"
    }
} else {
    Write-Info "已选择跳过 zipalign。"
}

# 签名方案配置
$enableV1 = $false; $enableV2 = $false; $enableV3 = $false; $enableV4 = $false
switch ($Schemes) {
    "all" { $enableV1=$true; $enableV2=$true; $enableV3=$true; $enableV4=$false } # V4 常用于增量分发，通常不必
    "v1"  { $enableV1=$true }
    "v2"  { $enableV2=$true }
    "v3"  { $enableV3=$true }
    "v4"  { $enableV4=$true } # 如需 V4，请同时确保下游支持
}

# 组装 apksigner 参数
$signArgs = @(
    "sign",
    "--ks", "$KeystorePath",
    "--ks-key-alias", "$Alias",
    "--ks-pass", "pass:$KeystorePass",
    "--key-pass", "pass:$KeyPass",
    "--v1-signing-enabled",  ($enableV1.ToString().ToLower()),
    "--v2-signing-enabled",  ($enableV2.ToString().ToLower()),
    "--v3-signing-enabled",  ($enableV3.ToString().ToLower()),
    "--v4-signing-enabled",  ($enableV4.ToString().ToLower()),
    "--out", "$signedApk",
    "$inputForSign"
)

Write-Info "执行签名..."
& $apksigner @signArgs | Write-Host
if ($LASTEXITCODE -ne 0 -or -not (Test-Path $signedApk)) {
    Write-Err "签名失败。"
    exit 1
}

# 验证签名
Write-Info "验证签名..."
& $apksigner verify --verbose --print-certs "$signedApk" | Write-Host
if ($LASTEXITCODE -ne 0) {
    Write-Err "签名验证失败。"
    exit 1
}

Write-Host ""
Write-Host "签名完成：" -ForegroundColor Green
Write-Host "  $signedApk" -ForegroundColor Green
```

使用示例
基本用法（自动查找最新 build-tools，v1+v2+v3，先 zipalign 再签名）：
> pwsh .\sign-apk.ps1 -ApkPath D:\app\reinforced.apk -KeystorePath D:\keys\release.jks -Alias myalias -KeystorePass 你的Keystore密码

指定 build-tools 目录与输出目录，并覆盖已有输出：
> pwsh .\sign-apk.ps1 -ApkPath D:\app\reinforced.apk -KeystorePath D:\keys\release.jks -Alias myalias -KeystorePass 你的Keystore密码 -KeyPass 你的Key密码 -BuildToolsDir "C:\Android\sdk\build-tools\35.0.0" -OutDir D:\out -Overwrite

某些加固包要求跳过对齐：
> pwsh .\sign-apk.ps1 -ApkPath D:\app\reinforced.apk -KeystorePath D:\keys\release.jks -Alias myalias -KeystorePass 你的Keystore密码 -SkipZipalign

只启用 v2 签名（极少数场景需要）：
> pwsh .\sign-apk.ps1 -ApkPath D:\app\reinforced.apk -KeystorePath D:\keys\release.jks -Alias myalias -KeystorePass 你的Keystore密码 -Schemes v2


**以上方法都需要每次输入Keystore密码和alias，也可以提前设置在脚本中配置好这些信息**
修改脚本最上面的param配置信息
```
param(
    [Parameter(Mandatory = $true)]
    [string]$ApkPath,                 # 待签名的APK

    [Parameter(Mandatory = $false)]
    [string]$KeystorePath = '签名文件路径',            # keystore 文件路径(.jks/.keystore)

    [Parameter(Mandatory = $false)]
    [string]$Alias = 'key别名',                   # key别名

    [Parameter(Mandatory = $false)]
    [string]$KeystorePass = '密码',            # keystore 密码

    [string]$KeyPass = '密码',                 # key 密码（如不填默认与 keystore 密码相同）

    [string]$BuildToolsDir,           # 指定 Android build-tools 目录（可选，如 C:\Android\sdk\build-tools\35.0.0）
    [string]$OutDir = '输出目录',                  # 输出目录（可选，默认与 APK 同目录）
)
```
这样就可以简化运行脚本为:
> pwsh 脚本路径(例如：.\cursor-sign1.ps1) -ApkPath 加固后的apk路径（例如：D:\work\xxx.apk）


前置条件
1. 已安装 Android SDK，并在 build-tools 中包含 apksigner.bat 和 zipalign.exe。
2. 或通过 -BuildToolsDir 明确指定 build-tools 路径。

如需同时产出未对齐签名、v4 签名或引入渠道信息，可告诉我你的要求，我可调整脚本。

***

## 脚本验证结论
进行一次简单的配置sdk、签名路径，之后每次加固后进行签名只用在PowerShell中执行脚本就可生成对应的签名后的apk文件。