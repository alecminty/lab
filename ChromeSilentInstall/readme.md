# Silent install of Chrome Browser on Windows Server 2016. #

**This is for lab testing when you do not want to deal with Microsoft Edge.**
Open Powershell on Windows 2016 server.
Enter the following command to download the Chrome browser, install it and make shortcut on the Desktop:

$Path = $env:TEMP; $Installer = "chrome_installer.exe"; Invoke-WebRequest "https://dl.google.com/chrome/install/latest/chrome_installer.exe" -OutFile $Path$Installer; Start-Process -FilePath $Path$Installer -Args "/silent /install" -Verb RunAs -Wait; Remove-Item $Path$Installer

