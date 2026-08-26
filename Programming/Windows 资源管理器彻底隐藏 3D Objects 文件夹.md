## 背景

在 Windows 10/11 中，系统默认在**此电脑（This PC）**和**左侧导航窗格（侧边栏）**中显示 **3D Objects（3D 对象）**文件夹。该文件夹主要供系统内置的 3D Viewer、Paint 3D 等应用存放模型文件，对大部分日常用户用处极少且占用视觉空间。

---

## 隐藏原理与注册表项

要彻底移除 3D Objects 文件夹（包括**此电脑主列表**与**左侧导航树**），需要修改 Windows 注册表中的相关项：

1. **“此电脑”主列表命名空间**：
   - 3D Objects 的 **GUID** 为：`{0DB7E03F-FC29-4DC6-9020-FF41B59E513A}`
   - 移除 64 位与 32 位视图下的命名空间挂载：
     - `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer\MyComputer\NameSpace\{0DB7E03F-FC29-4DC6-9020-FF41B59E513A}`
     - `HKLM\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Explorer\MyComputer\NameSpace\{0DB7E03F-FC29-4DC6-9020-FF41B59E513A}`
2. **文件夹显示策略（FolderDescriptions）**：
   - 将 **ThisPCPolicy** 键值设为 **Hide**：
     - `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer\FolderDescriptions\{31C0DD25-943C-4340-BE63-E3B902359857}\PropertyBag` -> `ThisPCPolicy` = `"Hide"`
     - `HKLM\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Explorer\FolderDescriptions\{31C0DD25-943C-4340-BE63-E3B902359857}\PropertyBag` -> `ThisPCPolicy` = `"Hide"`
3. **左侧导航窗格（侧边栏树状图）**：
   - 侧边栏是否显示由 **System.IsPinnedToNameSpaceTree** 和 **ShellFolder Attributes** 控制：
     - `HKLM\SOFTWARE\Classes\CLSID\{0DB7E03F-FC29-4DC6-9020-FF41B59E513A}` -> `System.IsPinnedToNameSpaceTree` = `0` (DWORD)
     - `HKLM\SOFTWARE\WOW6432Node\Classes\CLSID\{0DB7E03F-FC29-4DC6-9020-FF41B59E513A}` -> `System.IsPinnedToNameSpaceTree` = `0` (DWORD)
     - `HKCU\Software\Classes\CLSID\{0DB7E03F-FC29-4DC6-9020-FF41B59E513A}` -> `System.IsPinnedToNameSpaceTree` = `0` (DWORD)
     - `HKCU\Software\Classes\CLSID\{0DB7E03F-FC29-4DC6-9020-FF41B59E513A}\ShellFolder` -> `Attributes` = `0xf0104e60` (DWORD)

---

## 完整注册表脚本 (.reg)

### 彻底隐藏 3D Objects (.reg)

```reg
Windows Registry Editor Version 5.00

; 1. 从“此电脑”主列表移除命名空间
[-HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer\MyComputer\NameSpace\{0DB7E03F-FC29-4DC6-9020-FF41B59E513A}]
[-HKEY_LOCAL_MACHINE\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Explorer\MyComputer\NameSpace\{0DB7E03F-FC29-4DC6-9020-FF41B59E513A}]

; 2. 设置文件夹策略为 Hide
[HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer\FolderDescriptions\{31C0DD25-943C-4340-BE63-E3B902359857}\PropertyBag]
"ThisPCPolicy"="Hide"

[HKEY_LOCAL_MACHINE\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Explorer\FolderDescriptions\{31C0DD25-943C-4340-BE63-E3B902359857}\PropertyBag]
"ThisPCPolicy"="Hide"

; 3. 从左侧导航窗格（侧边栏）取消固定
[HKEY_LOCAL_MACHINE\SOFTWARE\Classes\CLSID\{0DB7E03F-FC29-4DC6-9020-FF41B59E513A}]
"System.IsPinnedToNameSpaceTree"=dword:00000000

[HKEY_LOCAL_MACHINE\SOFTWARE\WOW6432Node\Classes\CLSID\{0DB7E03F-FC29-4DC6-9020-FF41B59E513A}]
"System.IsPinnedToNameSpaceTree"=dword:00000000

[HKEY_CURRENT_USER\Software\Classes\CLSID\{0DB7E03F-FC29-4DC6-9020-FF41B59E513A}]
"System.IsPinnedToNameSpaceTree"=dword:00000000

[HKEY_CURRENT_USER\Software\Classes\CLSID\{0DB7E03F-FC29-4DC6-9020-FF41B59E513A}\ShellFolder]
"Attributes"=dword:f0104e60
```

### 恢复 3D Objects (.reg)

```reg
Windows Registry Editor Version 5.00

; 1. 恢复“此电脑”命名空间
[HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer\MyComputer\NameSpace\{0DB7E03F-FC29-4DC6-9020-FF41B59E513A}]
[HKEY_LOCAL_MACHINE\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Explorer\MyComputer\NameSpace\{0DB7E03F-FC29-4DC6-9020-FF41B59E513A}]

; 2. 恢复策略为 Show
[HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer\FolderDescriptions\{31C0DD25-943C-4340-BE63-E3B902359857}\PropertyBag]
"ThisPCPolicy"="Show"

[HKEY_LOCAL_MACHINE\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Explorer\FolderDescriptions\{31C0DD25-943C-4340-BE63-E3B902359857}\PropertyBag]
"ThisPCPolicy"="Show"

; 3. 恢复左侧导航窗格固定
[HKEY_LOCAL_MACHINE\SOFTWARE\Classes\CLSID\{0DB7E03F-FC29-4DC6-9020-FF41B59E513A}]
"System.IsPinnedToNameSpaceTree"=dword:00000001

[HKEY_LOCAL_MACHINE\SOFTWARE\WOW6432Node\Classes\CLSID\{0DB7E03F-FC29-4DC6-9020-FF41B59E513A}]
"System.IsPinnedToNameSpaceTree"=dword:00000001

[-HKEY_CURRENT_USER\Software\Classes\CLSID\{0DB7E03F-FC29-4DC6-9020-FF41B59E513A}]
```

---

## 自动化批处理脚本 (.bat)

修改完注册表后，需要**重启 Windows 资源管理器（explorer.exe）**使配置即时生效。可使用以下批处理脚本实现**自动提权并静默执行**：

```bat
@echo off
chcp 65001 >nul
net session >nul 2>&1
if %errorlevel% neq 0 (
    echo 正在请求管理员权限...
    powershell -Command "Start-Process '%~dpnx0' -Verb RunAs"
    exit /b
)

echo 正在从“此电脑”和“左侧导航窗格”中彻底移除 3D Objects 文件夹...

:: 1. 删除命名空间
reg delete "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer\MyComputer\NameSpace\{0DB7E03F-FC29-4DC6-9020-FF41B59E513A}" /f >nul 2>&1
reg delete "HKLM\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Explorer\MyComputer\NameSpace\{0DB7E03F-FC29-4DC6-9020-FF41B59E513A}" /f >nul 2>&1

:: 2. 设置策略
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer\FolderDescriptions\{31C0DD25-943C-4340-BE63-E3B902359857}\PropertyBag" /v "ThisPCPolicy" /t REG_SZ /d "Hide" /f >nul 2>&1
reg add "HKLM\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Explorer\FolderDescriptions\{31C0DD25-943C-4340-BE63-E3B902359857}\PropertyBag" /v "ThisPCPolicy" /t REG_SZ /d "Hide" /f >nul 2>&1

:: 3. 取消侧边栏固定
reg add "HKLM\SOFTWARE\Classes\CLSID\{0DB7E03F-FC29-4DC6-9020-FF41B59E513A}" /v "System.IsPinnedToNameSpaceTree" /t REG_DWORD /d 0 /f >nul 2>&1
reg add "HKLM\SOFTWARE\WOW6432Node\Classes\CLSID\{0DB7E03F-FC29-4DC6-9020-FF41B59E513A}" /v "System.IsPinnedToNameSpaceTree" /t REG_DWORD /d 0 /f >nul 2>&1
reg add "HKCU\Software\Classes\CLSID\{0DB7E03F-FC29-4DC6-9020-FF41B59E513A}" /v "System.IsPinnedToNameSpaceTree" /t REG_DWORD /d 0 /f >nul 2>&1
reg add "HKCU\Software\Classes\CLSID\{0DB7E03F-FC29-4DC6-9020-FF41B59E513A}\ShellFolder" /v "Attributes" /t REG_DWORD /d 4027596384 /f >nul 2>&1

echo 正在重启资源管理器以生效...
powershell -Command "Stop-Process -Name explorer -Force; Start-Sleep -Seconds 1; Start-Process explorer.exe" >nul 2>&1

echo 搞定！3D Objects 已完全从主界面和侧边栏中隐藏。
timeout /t 3 >nul
```
