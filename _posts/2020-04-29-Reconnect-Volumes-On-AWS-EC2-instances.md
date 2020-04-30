---
layout: post
title: "Reconnect Volumes On AWS EC2 instances."
date: 2020-04-29
summary: "When working with data disk on EC2 instances (or virtual machines) you do not want the restore the data when creating a new instance. This blog is about reconnecting the already available volumes in AWS with instances running on Windows."
minute: 8
---

Recently I have work on an AWS project. We had to create EC2 instances (virtual machines if you like Azure) running Windows server. The problem we ran into was that one of these machines had disks in a particular order with fixed disk drive letters. Also, we had to make sure the data on the drives were kept and could be reused when creating a new instance.

## EC2 Volume Manager

On AWS, you are not allowed to swop disks from one instance to another while the first one is still running, so you need a custom extension to disconnect the so you can reconnect them to the new instance. You can do this by installing the volume manager from my colleagues at Binx.io. You can find the EC2 volume manager over <a href ="https://binx.io/blog/2020/03/29/how-to-update-the-boot-image-of-a-stateful-machine-using-cloudformation/">here<a> with all the instructions you need to use it.

Unfortunately, this is not enough to restore the disk, including disk letters and system labels in windows, and will only connect the volumes to the ec2 instance. If the volume already has a partition, this will receive the default drive letter. 

## Get Instance information

To correctly restore the disks, we need to run a PowerShell in the EC2 user data option when the instance is booted for the first time. If you are using a tool like Packer (like we did), you can specify the volumes that are connected to the instance, and you can add PowerShell scripts to the user data option. When the instance is starts all the volumes defined in the packer file are connected. 

For restoring the disks in the correct order, we would like to create a generic PowerShell that can be reused for all machines or connect another volume. The script must therefore not have any knowledge about the machine it is running on. Second of all, we want it to be as explicit as possible without making any assumptions about settings.

The first step is we need to know what the instance is we are running the script on. We used Packer to create a base image; on this image, we installed chocolaty and used this to install the AWS command-line interface so we can use the AWS tooling. To get the instance information, AWS provides endpoints you can call to get the information you need. 

```powershell
 get the ID of the instance
$instance = Invoke-WebRequest -Uri http://169.254.169.254/latest/meta-data/instance-id -UseBasicParsing
```

## Get the connected volumes

Next, we need to determine what volumes are connected to this machine. To make sure we have all the volumes we need before creating the disks, we added Tags to the volumes, but these tags need to match the tag on the instance. For that, we aks the value of the tag on the instance called `ec2-volume-manager-attachment`.

```powershell
# get the volume of the tag which is connected to the volumes
$valuetag = Get-EC2Tag -Filter @{Name="resource-id";Value=$instance} | Where-Object {$_.Key -eq "ec2-volume-manager-attachment"  } | Select-Object -expand Value
```

On the volumes, we added the same tag; we use the value of this tag on the instance to get all the volumes in AWS with the same tag name and value. Now we know the number of volumes that should be connected to the instance before we start creating disks.

```powershell
# get the expected volume with the same tag / value pair
$expectedvolume = ((Get-EC2Volume).Tags | Where-Object { ($_.key -eq "ec2-volume-manager-attachment") -and ($_.value -eq $valuetag) }).Count
```
## Get the connected disks

When booting the instance, it may take a while before all the volumes are connected. So to make sure all the volumes are there, we can get all the volumes attached to the instance. 

Using the `get-ec2volume` and filter the on the `InstanceId` we found by calling the endpoint for the instance information. Adding another filter for the tags we get all the connected volumes.  

```powershell
# get the number of volumes connected to the instance
$disks = ((get-ec2volume) | Where-Object { ($_.Attachments.InstanceId -eq $instance) }).Tags | Where-Object { ($_.key -eq "ec2-volume-manager-attachment") -and ($_.value -eq $valuetag) }
```

Now we have the expected number of volumes and the number of connected volumes, if the numbers do not match we need to wait for a while until all the volumes are connected.  

```powershell
# if the numbers do not match, we are waiting for the volume to be attached to the instance
while ($disks.Count -ne $expectedvolume)
{
  $disks = ((get-ec2volume) | Where-Object { ($_.Attachments.InstanceId -eq $instance) }).Tags | Where-Object { ($_.key -eq "ec2-volume-manager-attachment") -and ($_.value -eq $valuetag) }
  Start-Sleep -s 5
  Write-Host "waiting for volumes..."
}
```

## Create disks from volumes

After all the volumes are attached to the instance, we can create disks in Windows. The very first time we create the disks, we can just call the disk management tools in Windows and clear and create the disks, but the second time we create an instance, the disks are already initialized, and there might be data on it. Remember, we need to keep the data on the disk, so we can not just clear and recreate the disk, this would cause all the data to be lost. There is a second problem with this approach, and that is the existing disk will get the default drive letter starting at D, and there is no guarantee that the volumes are connected in the same order when creating a new instance. And the last problem is we cannot just change the drive letter because the drive letter might already be taken. 

To solve all this, we need to implement a check to verify the disks we have, match them with the volumes we found and take them offline. But we only want to take offline these disks, not all the disks.  

We will get the volumes connected to the instance and use this information to get the disk matching the volume

```powershell
# get all the volumes 
$volumes = @(get-ec2volume) | Where-Object { ($_.Attachments.InstanceId -eq $instance) } | ForEach-Object { $_.VolumeId}
```

### take disks offline

We will loop through the found volumes and get the `volumeID` from the volume. This volumeID almost matches the disk serial number in Windows. We can call the `Get-Disk` command in Powershell and pass the `VolumeID` as the `SerialNumber`. If we find a disk we can take it offline, this will remove the drive from Windows and will unassign the drive letter.

```powershell
# Set all disk offline, because the will get a default driveletter
foreach ($vol in $volumes) 
{
  $volumeid = ((Get-EC2Volume -VolumeId $vol).VolumeId).Remove(0,4)
  
  $disk = Get-Disk | Where-Object {$_.SerialNumber -CLike "*$volumeid*"} 

  if ( ($disk.Number -ne 0) -and ($disk) )
  {
    Write-Host "Setting disknumber: "$disk.Number" offline - volume: $volumeid "
    Set-Disk -Number $disk.Number -IsOffline $True
  }
}
```

### Find the tags

After taking all the disks, except for the C-drive, offline, we can now add them one by one to our instance.

Again we loop through the volumes, but now we do not only get the volumeID but also the two tags we added to the volume, DriveLetter, and SystemLabel. We added these tags to the volumes in our Cloudformation templates. For each volume, this tells us what the drive letter is and what system label is we need to attach.  

```powershell
  Write-Host "Found volume with id: $volumeid"
  $DriveLetter = (Get-EC2Volume -VolumeId $vol).Tags | Where-Object { $_.key -eq "DriveLetter" } | Select-Object -expand Value
  $SystemLabel = (Get-EC2Volume -VolumeId $vol).Tags | Where-Object { $_.key -eq "SystemLabel" } | Select-Object -expand Value
  $disk = Get-Disk | Where-Object {$_.SerialNumber -CLike "*$volumeid*"} 
```

## Connect the disks

If we find all the three variables, we can create the disk in the instance. There are two situations; the first one is we did not create any partitions before, this is the very first time we create an instance with these volumes. Second is that the disks are already created once, and we need to reconnect them. We will start with the first option.

### Create a new disk

The disk we found has some properties we can use to determine the state. `PartitionStyle` tells us something about if the partition was already created. `OperationalStatus` tells us if the disk is offline or online. 

If the `PartitionStyle` equals `Raw` we did not create the disk before. So before we can use this disk we need to clear the disk and create a new Volume with the DriveLetter and SystemLabel we found before. 

```powershell
    if ( ($disk.PartitionStyle -eq "Raw") -and ($disk.OperationalStatus -eq "Offline") ) 
    {
        Initialize-Disk -Number $disk.Number 
        Clear-Disk -Number $disk.Number -RemoveData -Confirm:$false -PassThru
        Initialize-Disk -Number $disk.Number 
        New-Partition -DiskNumber $disk.Number -UseMaximumSize -DriveLetter $DriveLetter | Format-Volume -FileSystem NTFS -NewFileSystemLabel $SystemLabel
        Write-Host "Creating disk with DriveLetter $DriveLetter and SystemLabel $SystemLabel" 
    }
```

### Reconnect the existing disks

In the second option, the disk isn't `Raw` anymore but just offline. This means it has already been cleared before. First, we will just set the disk to `Online`, this will make the disk available in Windows. But we need this to match our DriveLetter and not the default drive letter. So we need to change the `DriveLetter` to the one we need. There is, however, another 'but' and that is we can only do this when the default drive letter does not match the drive letter we need if it already matches and we try to change it in the letter we need, it will raise an error. 

If it doesn't match, we can change the drive letter and set the system label. To do this, we need the DriveLetter we just found. By calling the `get-partition` and the `set-partion` we can change the drive letter to the one we need.

The last step is to set the system label by calling the `set-volume` method and passing the new drive letter and the SystemLabel. 

```powershell
    if ($disk.OperationalStatus -eq "Offline")
    {
        Set-Disk -Number $disk.Number -IsOffline $False
        $currentDrive = get-partition -DiskNumber $disk.Number| Where-Object { $_.Type -ne "Reserved" } | Select-Object -Expand DriveLetter
        if ( ($currentDrive -ne $DriveLetter) -and ($DriveLetter) -and ($currentDrive) )
        {
            Get-Partition -DriveLetter $currentDrive | Set-Partition -NewDriveLetter $DriveLetter
            Set-Volume -DriveLetter $DriveLetter -NewFileSystemLabel $SystemLabel
            LogWrite "Changing drive from $currentDrive to $DriveLetter"
        }
        LogWrite "Mounted disk with DriveLetter $DriveLetter and SystemLabel $SystemLabel"
    }
```

That's it, now we can use the disks in our instance. When other instances is booted, and this one is terminated, we will still have our data and the disk letter are the same, so the installer software will still work. 

You can find the complete demo with yaml files and the powershells <a href="https://github.com/binxio/ec2-volume-manager">here</a>:



