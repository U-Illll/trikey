# TriKey — 三重钥匙保险箱（制作方法）

一套「三个独立因子缺一不可」的个人加密保险箱方案：以 VeraCrypt 容器为箱体，把解锁链路拆成 **U 盘指纹 / 秘密文件 / 密码** 三层，任何单独一层泄露都无法开箱。

---

## 设计目标

- **单因子失效不等于整体失效**：U 盘丢了没用（没有密码）；密码泄露没用（没插 U 盘）；主机被彻底翻查也没用（箱体是加密的）
- **全离线**，无云端依赖、无联网验证
- **日常开箱一条命令**，用完全部卸载，箱体体积随时可扩

## 组件与角色

| 组件 | 示例位置 | 角色 |
|---|---|---|
| VeraCrypt 加密容器 | `D:\SecretVault.hc` | 箱体：强加密虚拟磁盘 |
| 授权 U 盘 | 任意 | 第一层：物理钥匙（以卷 UniqueId 识别） |
| 秘密文件 | `<U盘>\token\.tok` | 第二层：keyfile 因子（登记 SHA256） |
| 密码 | 只存脑子里 | 第三层：知识因子（PBKDF2-HMAC-SHA256，10 万次迭代） |
| 解锁脚本 | `C:\Tools\Unlock-TriKey.ps1` | 串联三层验证与挂载 |
| 校验数据目录 | `C:\ProgramData\TriKey\` | 只存各因子的「验证值」，不存任何明文密钥 |

---

## 一、准备箱体（VeraCrypt 容器）

1. 下载 VeraCrypt（便携版即可，放置于本机固定路径，如 `D:\VeraCrypt-x64.exe`）
2. 创建标准加密卷 `SecretVault.hc`，设置卷密码 —— **这个密码就是第三层密码**
3. 容器文件放在本机固定位置，与钥匙 U 盘分开保管

## 二、制作第一层：U 盘指纹

1. 选一个专用 U 盘（建议只当钥匙用，平时随身携带、与主机分离存放）
2. 插上后读取其卷 `UniqueId`：

```powershell
Get-Volume | Where-Object { $_.DriveType -eq 'Removable' } | Select-Object DriveLetter, UniqueId
```

3. 把该 `UniqueId` 原样写入 `C:\ProgramData\TriKey\dev_fingerprint.txt`
   （脚本运行时按此比对：**换了 U 盘就等于换了钥匙，需重新登记**）

## 三、制作第二层：秘密文件

1. 在 U 盘上建 `token\` 目录，放入一个 64 字节的随机秘密文件 `token\.tok`：

```powershell
$b = New-Object byte[] 64
[System.Security.Cryptography.RandomNumberGenerator]::Create().GetBytes($b)
[System.IO.File]::WriteAllBytes("<U盘>:\token\.tok", $b)
```

2. 计算其 SHA256 并登记（脚本比对哈希而非文件本身，**文件被替换或损坏即拒绝开箱**）：

```powershell
(Get-FileHash "<U盘>:\token\.tok" -Algorithm SHA256).Hash | Set-Content "C:\ProgramData\TriKey\token_hash.txt"
```

3. 建议将该文件设为隐藏 + 只读（降低误删风险）

## 四、制作第三层：密码验证数据

只保存「盐 + 派生值」，绝不保存密码本身：

```powershell
# 1) 生成 16 字节随机盐
$salt = New-Object byte[] 16
[System.Security.Cryptography.RandomNumberGenerator]::Create().GetBytes($salt)
[System.IO.File]::WriteAllBytes("C:\ProgramData\TriKey\pw_salt.bin", $salt)

# 2) 输入密码 → PBKDF2 派生 32 字节 → Base64 存储
$pw = Read-Host "设置三重解锁密码" -AsSecureString
$d = New-Object System.Security.Cryptography.Rfc2898DeriveBytes -ArgumentList ($pw, $salt, 100000)
[System.Convert]::ToBase64String($d.GetBytes(32)) | Set-Content "C:\ProgramData\TriKey\pw_hash.txt"
```

## 五、部署解锁脚本

把下面的脚本保存为 `C:\Tools\Unlock-TriKey.ps1`（脚本以管理员权限运行），可配桌面快捷方式：

```
powershell -ExecutionPolicy Bypass -File "C:\Tools\Unlock-TriKey.ps1"
```

**脚本模板**（路径可按需修改；此处为通用化版本）：

```powershell
#Requires -RunAsAdministrator
$ErrorActionPreference = "Stop"
$configPath  = "C:\ProgramData\TriKey"     # 校验数据目录
$tokenRelative = "token\.tok"               # U 盘内秘密文件相对路径
$containerPath = "D:\SecretVault.hc"        # VeraCrypt 容器
$mountDrive = "M"                           # 挂载盘符
$veraCrypt = "D:\VeraCrypt-x64.exe"

# 第一层：U 盘指纹
$storedFinger = (Get-Content "$configPath\dev_fingerprint.txt").Trim()
$usbDriveLetter = $null
foreach ($vol in (Get-Volume | Where-Object { $_.DriveType -eq 'Removable' })) {
    if ($vol.UniqueId -eq $storedFinger) { $usbDriveLetter = $vol.DriveLetter; break }
}
if (-not $usbDriveLetter) { Write-Host "验证失败：授权 U 盘未插入" -ForegroundColor Red; exit 1 }

# 第二层：秘密文件
$tokenPath = Join-Path "${usbDriveLetter}:" $tokenRelative
if (-not (Test-Path $tokenPath)) { Write-Host "验证失败：秘密文件缺失" -ForegroundColor Red; exit 1 }
$currentHash = (Get-FileHash $tokenPath -Algorithm SHA256).Hash
$storedHash = (Get-Content "$configPath\token_hash.txt").Trim()
if ($currentHash -ne $storedHash) { Write-Host "验证失败：秘密文件已损坏或被替换" -ForegroundColor Red; exit 1 }

# 第三层：密码
$securePw = Read-Host "请输入三重解锁密码" -AsSecureString
$salt = Get-Content "$configPath\pw_salt.bin" -Encoding Byte -Raw
$deriveBytes = New-Object System.Security.Cryptography.Rfc2898DeriveBytes -ArgumentList ($securePw, $salt, 100000)
$inputHash = $deriveBytes.GetBytes(32)
$storedHashBytes = [System.Convert]::FromBase64String((Get-Content "$configPath\pw_hash.txt").Trim())
if (-not ([System.Linq.Enumerable]::SequenceEqual([byte[]]$inputHash, [byte[]]$storedHashBytes))) {
    Write-Host "验证失败：密码错误" -ForegroundColor Red; exit 1
}

# 三层通过 → 挂载
$BSTR = [System.Runtime.InteropServices.Marshal]::SecureStringToBSTR($securePw)
$plainPw = [System.Runtime.InteropServices.Marshal]::PtrToStringAuto($BSTR)
& $veraCrypt /volume $containerPath /letter $mountDrive /password $plainPw /quit /silent
[System.Runtime.InteropServices.Marshal]::ZeroFreeBSTR($BSTR)

if (Test-Path "${mountDrive}:\") {
    Write-Host "保险箱已挂载至 ${mountDrive}: ，安全操作后请及时卸载。" -ForegroundColor Green
} else {
    Write-Host "挂载失败，请检查容器路径或密码" -ForegroundColor Red
}
```

## 六、日常使用流程

1. 插入授权 U 盘
2. 运行解锁脚本 → 依次通过 **U 盘指纹 → 秘密文件 → 密码** 三层校验
3. 容器挂载到指定盘符（示例 `M:`），正常读写
4. **用完立即卸载**（VeraCrypt 托盘图标 → 卸载，或断开盘符）——盘符消失，箱体回到加密状态

---

## 安全注记

- 校验数据目录（`dev_fingerprint` / `token_hash` / `pw_salt` / `pw_hash`）即使被完整读取，也**不足以**开箱：前两者只包含 U 盘与文件的存在性证明，密码层是 10 万次迭代的 PBKDF2 派生值，无法反推
- 方案的真正强度在**物理分离**：U 盘与主机分开保管、密码不写入任何设备——三者同时在手才可开
- VeraCrypt 容器即使被整体拷走，仍是强加密状态（暴力破解成本取决于密码强度）
- 本方案面向**个人合法数据**保护；VeraCrypt 为开源软件；请遵守所在地法律法规

---

*本文档为制作方法说明（通用化模板）。所有指纹、哈希、路径均为示例，不含任何真实凭据。*
