# 1. Cài Wget
winget install JernejSimoncic.Wget --accept-source-agreements --accept-package-agreements

# 2. Tạo folder làm việc và chui vào
New-Item -ItemType Directory -Force -Path "C:\macos27-site"
Set-Location "C:\macos27-site"

# 3. Tìm đường dẫn thực tế của wget.exe vừa cài để gọi trực tiếp (không sợ lỗi PATH/Alias)
$wgetPath = (Get-Command wget.exe -ErrorAction SilentlyContinue).Path
if (-not $wgetPath) {
    $wgetPath = (Get-ChildItem -Path "$env:ProgramFiles", "${env:ProgramFiles(x86)}", "$env:LOCALAPPDATA\Microsoft\WinGet" -Filter "wget.exe" -Recurse -ErrorAction SilentlyContinue | Select-Object -First 1).FullName
}

# 4. Tiến hành cào web
& $wgetPath --mirror --convert-links --adjust-extension --page-requisites --no-parent --user-agent="Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36" https://macos27.kimi.page/
