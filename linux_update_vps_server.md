# Update a Linux VPS running Ubuntu

## 1. Connect to the VPS

## 2. Update package lists: 
```
sudo apt update
```
This will check for available updates

## 3. Upgrade installed packages
```
sudo apt upgrade -y
```
Install the packages with updates

## 4. Clean up the update files
```
sudo apt autoremove
```
Remove the unneccessary packages

## 5. Reboot (if needed) 
```
sudo reboot
```
If the kernel was updated then we neeed to reboot our VPS

