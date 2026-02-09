References:
https://learn.microsoft.com/en-us/windows/wsl/
https://learn.microsoft.com/en-us/windows/wsl/install
https://learn.microsoft.com/en-us/windows/wsl/install-manual
https://learn.microsoft.com/en-us/windows/wsl/setup/environment
https://apps.microsoft.com/detail/9n0dx20hk701?hl=en-US&gl=US
https://learn.microsoft.com/en-us/windows/wsl/tutorials/wsl-vscode
https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.vscode-remote-extensionpack
https://learn.microsoft.com/en-us/windows/wsl/disk-space
https://learn.microsoft.com/en-us/windows/wsl/tutorials/linux
https://learn.microsoft.com/en-us/windows/wsl/tutorials/gpu-compute (Seems interesting TODO: when necessity arises)
https://learn.microsoft.com/en-us/windows/wsl/networking
https://learn.microsoft.com/en-us/windows/wsl/tutorials/wsl-containers 

1. Verify windows version
saettings -> about
Windows version is 2004
Build version 1904 or higher

(Skip to step 4 during automated deployment)
2. Turn windows features on or off
Make sure virtulization is enabled in BIOS
Enable:
    (Version locked, available only on windows enterprise and pro)
    containers
    Hyper-V

    Virtual Machine Platform
    Windows Hypervisor Platform
    Windows Subsystem for Linux

3. Download and install windows terminal
https://apps.microsoft.com/detail/9n0dx20hk701?hl=en-US&gl=US


4. Enable Windows Subsystem for linux
Powershell administrator terminal:
    dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart

5. Enable Virtual Machine Feature
Powershell administrator terminal:
    dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart

6. Download the Linux kernel update package
    wsl.exe --install

7. (optional - Update if already exists) Update wsl
    wsl.exe --update

8. Set WSL 2 as your default version
    wsl --set-default-version 2

9. List available linux distributions
    wsl.exe --list --online
    To view local installed distributions
    wsl --list 

10. Specify username and password for distro when prompted

11. (optional) Update password for a specific distro
Powershell administrator terminal:
    wsl -d <DistroName> -u root

12. Update WSL
WSL terminal:
    sudo apt-get update

13.
    sudo apt-get install wget ca-certificates

14. Get the git prompt scripts
    curl https://raw.githubusercontent.com/git/git/master/contrib/completion/git-prompt.sh -o ~/.git_prompt.sh
    
    curl https://raw.githubusercontent.com/git/git/master/contrib/completion/git-completion.bash -o ~/.git_completion.sh

15. Run sed commands
    sed -i "$ s/$/\n\n# Set prompt to show the git branch name and enable git completions/" ~/.bashrc
    
    sed -i "$ s/$/\nsource ~\/.git_prompt.sh/" ~/.bashrc
    
    sed -i "$ s/$/\nsource ~\/.git_completion.sh/" ~/.bashrc
    
    sed -i "$ s/$/\nPS1='\\\[\\\e[0;32m\\\]\\\u\\\[\\\e[m\\\]:\\\[\\\e[0;36m\\\]\\\w\\\[\\\e[m\\\]\\\[\\\e[0;35m\\\]\$\(__git_ps1 \" \(%s\)\"\)\\\[\\\e[m\\\]$ '/" ~/.bashrc

16. Reload bash config 
    source ~/.bashrc