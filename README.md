# OpenFreezeCenter (OFC)

> [!NOTE]
> This repository is an improved fork of the original [OpenFreezeCenter](https://github.com/YoCodingMonster/OpenFreezeCenter) project.

- Provides a UI and automated scripts in order to control MSI Laptops. Check the #Supported section to see what models are supported.
- Made for Linux, as MSI does not have a native Linux client.
- if you do now want to run the GUI or if it is not working for you then try
  # OpenFreezeCenter-Lite (OFC-l)
  - Same thing just without GUI
  - https://github.com/YoCodingMonster/OpenFreezeCenter-Lite

# INSTALLATION / UPDATING
- ```cd``` into the download folder and execute (UBUNTU)
  - ```chmod +x file_1.sh```
  - ```chmod +x file_2.sh```
  - ```chmod +x install.sh```
- Now run the ```install.sh```, That will install all the dependencies and create a virtual python environment on desktop for the script to work.
- (ONLY FOR INSTALLATION) ```Reboot``` after the script complete the first run.

# RUNNING
- Run ```install.sh``` from the desktop folder. 

## Creating an Application Launcher Shortcut (Manjaro & Other Distros)
To launch OpenFreezeCenter directly from your desktop menus, panels, or shortcuts without opening a terminal, you can create a custom Application Launcher:

1. Create a custom desktop entry (e.g., a file named `OFC.desktop` in `~/.local/share/applications/` or on your Desktop).
2. Set the **Exec** / Command field to the following:
   ```bash
   bash -c "password=\$(zenity --password --title='OFC Admin Authentication') && echo \$password | sudo -S env DISPLAY=\$DISPLAY XAUTHORITY=\$XAUTHORITY /home/sjs/Desktop/OFC/bin/python /home/sjs/Desktop/OFC/OFC.py"
   ```
3. Set the launcher to execute. This command uses `zenity` to securely prompt you for your admin password via a GUI dialog, passing it to `sudo` to run the application with root privileges inside the `uv` virtual environment.

> [!NOTE]
> **Zenity Installation**: If `zenity` is not installed on your system, install it using your package manager:
> - **Manjaro / Arch**: `sudo pacman -S zenity`
> - **Ubuntu / Debian**: `sudo apt install zenity`
> - **Fedora / Red Hat**: `sudo dnf install zenity`

## Supported Laptop models (tested)
- MSI GP76 11UG
- MSI GF63 Thin 11SC

## Supported Linux Distro (tested)
- Ubuntu

## Issue format
- ISSUE # [CPU] - [LAPTOP MODEL] - [LINUX DISTRO]
  - ```Example``` ISSUE # i7-11800H - MSI GP76 11UG - UBUNTU 23.05

## Feedback
- Please provide suggestions under the Feedback discussion tab!

## Goals
- [X] Fan Control GUI
- [X] Basic temperature and RPM monitoring
- [ ] Advanced & Basic GUI control
- [X] Battery Threshold
- [ ] Webcam control
