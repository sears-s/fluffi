<!---
Copyright 2017-2020 Siemens AG

Permission is hereby granted, free of charge, to any person obtaining a
copy of this software and associated documentation files (the
"Software"), to deal in the Software without restriction, including without
limitation the rights to use, copy, modify, merge, publish, distribute,
sublicense, and/or sell copies of the Software, and to permit persons to whom the
Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be
included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS
OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT
SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR
OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE,
ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER
DEALINGS IN THE SOFTWARE.

Author(s): Pascal Eckmann
-->

# Microsoft Windows

## Prepare WinPE image
1. Install the [Windows Assessment and Deployment Kit (ADK)](https://docs.microsoft.com/en-us/windows-hardware/get-started/adk-install) (preferably on a freshly installed Windows) with standard settings
    - Install ADK
    - Install ADK-Windows PE-Addon 
2. Run `Windows Kit/Deployment and Imaging Tools Environment` from the start menu to open the CLI ...
3. ... and execute the following commands step-by-step to create Windows PE
    - `copype.cmd amd64 C:\winpe_amd64`
    - `dism /mount-image /imagefile:c:\winpe_amd64\media\sources\boot.wim /index:1 /mountdir:C:\winpe_amd64\mount`
    - `xcopy c:\winpe_amd64\mount\windows\boot\pxe\*.* c:\PXE\boot\`
    - `copy C:\winpe_amd64\media\boot\boot.sdi c:\PXE\boot\`
    - `copy C:\winpe_amd64\media\sources\boot.wim c:\PXE\boot\`
    - `bcdedit /createstore C:\PXE\BCD`    
    _-> maybe use the other command line to get another version of bcedit_
    - `bcdedit /store c:\PXE\BCD /create {ramdiskoptions} /d "Ramdisk options"`
    - `bcdedit /store C:\PXE\BCD /set {ramdiskoptions} ramdisksdidevice boot`
    - `bcdedit /store C:\PXE\BCD /set {ramdiskoptions} ramdisksdipath \boot\boot.sdi`
    - `bcdedit /store C:\PXE\BCD /create /d "winpe boot image" /application osloader`    
    _-> use the GUID resulted in here, e.g. `bcdedit /store C:\PXE\BCD /set {00000000-0000-0000-0000-000000000000} device [...]`_
    - `bcdedit /store C:\PXE\BCD /set {GUID} device ramdisk=[boot]\boot\boot.wim,{ramdiskoptions}`
    - `bcdedit /store C:\PXE\BCD /set {GUID} path \windows\system32\winload.exe`
    - `bcdedit /store C:\PXE\BCD /set {GUID} osdevice ramdisk=[boot]\boot\boot.wim,{ramdiskoptions}`
    - `bcdedit /store C:\PXE\BCD /set {GUID} systemroot \windows`
    - `bcdedit /store C:\PXE\BCD /set {GUID} detecthal Yes`
    - `bcdedit /store c:\PXE\BCD /set {GUID} winpe Yes`
    - `bcdedit /store C:\PXE\BCD /set {GUID} device ramdisk=[boot]\boot\boot.wim,{ramdiskoptions}`
    - `bcdedit /store C:\PXE\BCD /set {GUID} path \windows\system32\winload.exe`
    - `bcdedit /store C:\PXE\BCD /set {GUID} osdevice ramdisk=[boot]\boot\boot.wim,{ramdiskoptions}`
    - `bcdedit /store C:\PXE\BCD /set {GUID} systemroot \windows`
    - `bcdedit /store C:\PXE\BCD /set {GUID} detecthal Yes`
    - `bcdedit /store C:\PXE\BCD /set {GUID} winpe Yes`
    - `bcdedit /store C:\PXE\BCD /create {bootmgr} /d "boot manager"`
    - `bcdedit /store C:\PXE\BCD /set {bootmgr} timeout 30`
    - `bcdedit /store C:\PXE\BCD -displayorder {GUID} -addlast`
4. Manipulating the `boot.wim` of the new Windows PE image
	- Mount the image
	    - Execute: `mkdir C:\mount\boot\`
	    - Execute: `dism /Mount-Wim /wimfile:C:\PXE\boot\boot.wim /index:1 /MountDir:C:\mount\boot\`
	- Navigate to `C:\mount\boot\Windows\System32\` and add/change a file `Startnet.cmd` with the following content, based on your settings (note: [windows] is a placeholder for your folder containing the windows image on the SMB share):
		```
		wpeinit
        Wpeutil initializenetwork
        Wpeutil WaitForNetwork
		net use Y: \\250.154.54.178\install\[windows] /user:nobody pass
		Y:
		setup.exe /Unattend:Y:\unattend.xml
		```
	- Typically you would use the address `smb.fluffi` here (`net use Y: \\smb.fluffi\install\[windows] /user:nobody pass`), but it always results in inconsistent behavior. Therefore the IP address of the SMB share must be used here
    - Unmount the image and commit changes by executing `dism /Unmount-Wim /MountDir:C:\mount\boot /commit`
    
## Configure Windows for unattended installation
- To automate the Windows installation an `unattend.xml` file should be created and copied in the [windows] directory on the SMB share, where `setup.exe` is.
- You can use the provided `unattend.xml` files for [Windows Server 2008](windows/windowsServer2008/unattend.xml), [Windows Server 2016](windows/windowsServer2016/unattend.xml) or [Windows Server 2019](windows/windowsServer2019/unattend.xml)
    - Change [username] to the same which you defined for _ansible_user_ and [password] to the same as _ansible_password_ in the section _[windows:vars]_ in your [hosts](../../srv/fluffi/data/polenext/projects/1/hosts) file for Polemarch, and [key] to your Windows product key
- To create these files, use one of the many instructions which are available online, like this [one](https://www.virtualizationhowto.com/2019/05/create-unattend-answer-file-for-windows-server-2019-automated-packer-installation/) 
    - For automatic setup of the hostname, insert this into your `unattend.xml` in `<settings pass="oobeSystem">` as own component and turn AutoLogon on (`<LogonCount>1</LogonCount>`)   
        ```
        <FirstLogonCommands>
            <SynchronousCommand wcm:action="add">
                <CommandLine>cmd.exe /c net use y: \\250.154.54.178\install\initial /user:nobody pass</CommandLine>
                <Order>1</Order>
                <Description>Mount Share</Description>
            </SynchronousCommand>
            <SynchronousCommand wcm:action="add">
                <CommandLine>cmd.exe /C start /wait y:\initialConfiguration.bat</CommandLine>
                <Description>Execute initial configuration script</Description>
                <Order>2</Order>
            </SynchronousCommand>
            <SynchronousCommand wcm:action="add">
                <CommandLine>cmd.exe /c net use y: /Delete /yes</CommandLine>
                <Description>unmout Drive</Description>
                <Order>3</Order>
            </SynchronousCommand>
        </FirstLogonCommands>
        ``` 
    - Use the prepared `unattend.xml` files as cheat sheet
    
## Download Windows
- Download a Windows Server Image from Microsoft
- Extract the downloaded image
- Add your prepared unattend.xml file to the extracted data

## Copy Windows image to infrastructure
- Copy Windows data to SMB share
    ```
    install 
    ├── [windows]
    │   ├── boot
    │   │   └── ...
    │   ├── efi
    │   │   └── ...
    │   ├── sources
    │   │   └── ...
    │   ├── ...
    │   │   └── ...
    │   ├── unattend.xml
    │   └── ...
    ├── another_os
    │   └── ...
    └── ...
    ```
- Note: Set all permissions for directories and files on the SMB share to 777
   
## Copy Windows PE image to infrastructure
- Create a folder in `ftp.fluffi/tftp-roots/`, e.g. `ftp.fluffi/tftp-roots/windows` 
- Copy all PXE start files from `C:\PXE\boot\` to `ftp://ftp.fluffi/tftp-roots/windows`
- Create two folders, `boot` and `Boot`, inside `windows`
    - Copy the following files into `boot` and `Boot`
        - `boot.sdi`
        - `boot.wim`
        - `BCD` (from `C:\PXE\`)
- Your `tftp-roots` should now look like this
    ```
    tftp-roots 
    ├── windows
    │   ├── boot
    │   │   ├── boot.sdi
    │   │   ├── boot.wim
    │   │   └── BCD
    │   ├── Boot
    │   │   ├── boot.sdi
    │   │   ├── boot.wim
    │   │   └── BCD
    │   ├── abortpxe.com
    │   ├── bootmgr.exe
    │   ├── ...
    │   └── wdsnbp.com
    ├── another_os
    │   └── ...
    └── ...
    ```
