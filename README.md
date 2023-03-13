# Hyper-V for rapid kernel and driver debugging
The following steps will guide you through the process of creating a Windows kernel debugging VM along with some scripts to spin everything up within a few seconds. It is based off of my other rapid kernel debugging setup guides, however, it's much faster, doesn't require a physical volume and is generally much more stable.

## Prerequisites
* Hyper-V
* Windows 10/11 image file - I personally use Windows 10 Pro, but any version should work

## Creating the VM
1. Create the VM as you normally would. It is recommended keep all hardware requirements to a minimum. Do not start the VM yet
2. Optional: If you're using Windows 11, make sure Secure Boot is **disabled**
3. Optional: Enable "Guest Services". I'm not sure if this is required or if it gives us any benefits, but I usually enable it regardless
4. Open the VM settings
5. Under "SCSI Controller", add another Hard Drive. You may call this file whatever you wish, I chose to name mine "Kernel Development Data". Remember the path of the .vhdx file, you'll need it later

You are now ready to prepare Windows. We'll use the additional volume/disk to copy data from host to guest while the VM is shut down.

## Preparing Windows
1. Install Windows as you normally would. It is recommended to license the VM
3. Install all pending updates
4. Force Windows to ignore all startup and shutdown failures:
```batch
bcdedit /set {current} bootstatuspolicy ignoreallfailures
```
5. Disable integrity checks and enable test signing
```batch
bcdedit /set testsigning on
bcdedit /set nointegritychecks on
```
6. Disable UAC via registry editor:
```
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System > EnableLUA = 0
```
7. Enable kernel debugging. Change `$HOSTIP` with your hosts IP address. Make sure the VM can ping the IP. `$HOSTPORT` should be replaced with the port that WinDbg will be listening on (default is 55000):
```batch
bcdedit /debug on
bcdedit /dbgsettings net hostip:$HOSTIP port:$HOSTPORT key:1.1.1.1
```
8. _Optional_: Allow the VM to redirect all debug messages to WinDbg. Create the key "Debug Print Filter" if it doesn't exist (you can also create a DWORD entry for a specific service/driver if you just want to forward those):
```
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Session Manager\Debug Print Filter > DEFAULT = 0xFFFFFFFF
```
9. Open the "Disk Management" utility and format the previously created .vhdx to NTFS or FAT32. Assign a specific letter to the volume (for example `K:\`)
10. Create a service for our driver and a scheduled task to load the driver on startup. I generally recommend to use a generic name for the driver and then rename your driver to the generic name before deploying it to the VM:
```batch
sc create layle binPath= "K:\layle.sys" type= kernel
schtasks /create /sc onstart /tr "C:\onboot.bat" /tn driveronboot /ru SYSTEM /f
```
11. Create the file `C:\onboot.bat` with the following content. Again, make sure to replace `$HOSTIP` and `$HOSTPORT` (the first line may not necessarily be needed, but I found that sometimes the settings would be deleted under rare circumstances):
```batch
bcdedit /dbgsettings net hostip:$HOSTIP port:$HOSTPORT key:1.1.1.1
sc start layle
```
12. Optional: Remove the user's password to avoid having to enter it every time you boot the VM
13. Optional: Disable Windows Defender via GPO

Finally, shutdown your VM. It is now ready to be used for kernel and driver debugging.

## Automating driver deployment
Chances are that you're working on a driver and you'd like to build the project, copy the files, spin up the VM along with WinDbg and connect to the VM via RDP. The following PowerShell script can automate the process:

```powershell
# Your VM name and path to the additional data volume
$VHD = "C:\ProgramData\Microsoft\Windows\Virtual Hard Disks\Kernel Development Data.vhdx"
$Name = "Kernel Development"

# Your build steps. You'll likely want to modify these
cd .\build; ninja; cd ..

# Mount the disk, and grab the drive letter
$DriveLetter = (Mount-VHD -Path $VHD -PassThru | Get-Disk | Get-Partition | Get-Volume).DriveLetter

# Copy your files to the disk; tweak accordingly
cp .\build\pphv.sys "$($DriveLetter):\"
cp .\build\pphv-um.exe "$($DriveLetter):\"

# Unmount the disk, start the VM, connect to it and launch WinDbg
Dismount-VHD -Path $VHD
Start-VM -Name $Name
$VM = Get-VM -Name "Kernel Development"
vmconnect.exe $env:COMPUTERNAME $VM.Name -G $VM.Id
WinDbgX -k net:port=55000,key=1.1.1.1

# Shutdown the VM as soon as you close WinDbg
Stop-VM -Name $Name -TurnOff
```

## Credits
The idea is based off of [this](https://secret.club/2020/04/10/kernel_debugging_in_seconds.html) article. Some information has been directly copied from the article. The project has been adapter for Hyper-V based on my other 2 setup guides for [VMware Workstation](https://github.com/ioncodes/kdbg-driver-workstation) and [Vagrant](https://github.com/ioncodes/kdbg-driver-vagrant).
