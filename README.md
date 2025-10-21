# VS Code: Lightweight Remote Sync Workflow

This template provides a configuration for a lightweight, `rsync`-based remote development workflow in Visual Studio Code.

It is designed for situations where using the official **Remote - SSH** extension is not feasible due to resource constraints on the server. Instead of running a VS Code server remotely, this setup keeps your work environment entirely on your local machine and uses `rsync` to efficiently sync files to and from a remote server.

### ✨ Features

  * **Push on Save**: Automatically syncs your local changes to the server the moment you save a file.
  * **Manual Pull Task**: A keyboard shortcut (`Ctrl+Alt+R`) to pull changes (like logs or results) from the server to your local machine.
  * **Directional Ignore Files**: Use `.rsyncignore-push` and `.rsyncignore-pull` to control exactly which files to ignore in each direction.
  * **Silent Operation**: Tasks run silently in the background, only showing the terminal if an error occurs.
  * **Lightweight**: Puts virtually no load on the remote server outside of the brief `rsync` connections.

-----

## SETUP: ONE-TIME STEPS

### 1\. Install Prerequisites

Make sure you have the following software installed.

  * **Local Machine (Windows/WSL)**:

    1.  **Windows Subsystem for Linux (WSL)**: If not installed, open PowerShell as Administrator and run `wsl --install`.
    2.  **Visual Studio Code**: [Download here](https://code.visualstudio.com/).
    3.  **VS Code WSL Extension**: Install the official **WSL** extension from the VS Code Marketplace. [Link here](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-wsl).
    4.  **VS Code `runonsave` Extension**: Install the **Run on Save** extension. [Link here](https://marketplace.visualstudio.com/items?itemName=emeraldwalk.RunOnSave).
    5.  **`rsync` in WSL**: Open your WSL terminal (e.g., Ubuntu) and run:
        ```bash
        sudo apt-get update && sudo apt-get install rsync
        ```

  * **Remote Server** (`gpulab` in this example):

    1.  **`rsync`**: Connect to your server and ensure `rsync` is installed.
        ```bash
        # On Debian/Ubuntu
        sudo apt-get update && sudo apt-get install rsync
        ```

### 2\. Set Up Passwordless SSH Access

This allows `rsync` to connect without asking for a password every time.

1.  **Generate an SSH Key (in WSL)**: If you don't already have one, open your WSL terminal and run:

    ```bash
    ssh-keygen -t rsa -b 4096
    ```

    Press Enter to accept the defaults.

2.  **Copy Your Public Key to the Server**: Run the following command, replacing `user@gpulab` with your actual username and server address. It will ask for your password one last time.

    ```bash
    ssh-copy-id user@gpulab
    ```

3.  **Configure an SSH Alias**: Create or edit the SSH config file to create a simple alias for your server.

    ```bash
    # Open the config file in a text editor
    nano ~/.ssh/config
    ```

    Add the following entry, replacing the values for `HostName` and `User`.

    ```
    Host gpulab
      HostName your-server-address.com
      User your_username
      IdentityFile ~/.ssh/id_rsa
    ```

    Save the file (`Ctrl+O`, Enter, `Ctrl+X`). You should now be able to connect by just typing `ssh gpulab`.

### 3\. Connect VS Code to WSL

To ensure `rsync` and `ssh` commands work correctly, you must run VS Code in the context of your WSL environment.

1.  Open VS Code.
2.  Click the green icon in the bottom-left corner of the window.
3.  Select **"New WSL Window"** from the command palette.
4.  A new VS Code window will open, connected to your WSL instance. Use this window to open your project folder.

-----

## 🚀 PROJECT CONFIGURATION

Clone this repository or place these files in your project's root directory. The paths in the configuration files assume your local project is at `/mnt/c/Users/matya/source/repos/folder-sync/` and the remote is at `/home/brabecm4/phd/folder-sync/`. **Remember to change these paths to match your setup.**

### Directory Structure

```
.
├── .rsyncignore-pull
├── .rsyncignore-push
├── .vscode
│   ├── settings.json
│   └── tasks.json
├── README.md
└── test.py
```

### Configuration Files

#### `.vscode/settings.json`

This file configures the "push on save" behavior.

```json
{
    "emeraldwalk.runonsave": {
        "commands": [
            {
                "match": ".*",
                "cmd": "rsync -avz --exclude-from='.rsyncignore-push' /mnt/c/Users/matya/source/repos/folder-sync/ gpulab:/home/brabecm4/phd/folder-sync/"
            }
        ]
    },
    "terminal.integrated.profiles.linux": {
        "Remote SSH (gpulab)": {
            "path": "ssh",
            "args": [
                "gpulab",
                "-t",
                "cd /home/brabecm4/phd/folder-sync && bash"
            ]
        }
    },
    "terminal.integrated.defaultProfile.linux": "Remote SSH (gpulab)"
}
```

#### `.vscode/tasks.json`

This file configures the manual "pull" task, which you can run with a keyboard shortcut.

```json
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "Pull from Server",
            "type": "shell",
            "command": "rsync -avz --exclude-from='.rsyncignore-pull' gpulab:/home/brabecm4/phd/folder-sync/ /mnt/c/Users/matya/source/repos/folder-sync/",
            "problemMatcher": [],
            "presentation": {
                "reveal": "silent"
            },
            "options": {
                "shell": {
                    "executable": "/bin/bash",
                    "args": ["-c"]
                }
            }
        }
    ]
}
```

#### `.vscode/keybindings.json` (Optional but Recommended)

To set up the `Ctrl+Alt+R` shortcut, open the command palette (`Ctrl+Shift+P`), find **`Preferences: Open Keyboard Shortcuts (JSON)`**, and add:

```json
{
    "key": "ctrl+alt+r",
    "command": "workbench.action.tasks.runTask",
    "args": "Pull from Server"
}
```

#### `.rsyncignore-push`

List of files/folders to **ignore when uploading** to the server.

```
# Don't upload VS Code workspace settings
.vscode/

# Ignore local-only files
my_local_notes.md
*.tmp
```

#### `.rsyncignore-pull`

List of files/folders to **ignore when downloading** from the server.

```
# Ignore large datasets or server-side logs
big_dataset/
server_logs/
*.csv
```

-----

## ⚠️ A Note on Using the `--delete` Flag

The `rsync --delete` flag is a powerful tool that makes the destination an exact mirror of the source. However, it can be dangerous in a two-way sync setup like this one. For example, if you pull from the server, a local file that doesn't exist on the server will be deleted.

For safety, the `--delete` flag has been **removed** from the default configuration.

If you want to use it safely, the best practice is to **separate your source code from your results**.

  * **Source Code Directory (`src/`)**: This folder is synced from your local machine to the server. You can safely use `--delete` in your **push** command for this folder.
  * **Results Directory (`results/`)**: This folder contains experiment outputs, logs, and other files generated on the server. You would only sync this from the server to your local machine. You can safely use `--delete` in your **pull** command for this folder.

This separation prevents source code from being deleted by a result-pulling task and vice-versa.

## WORKFLOW: HOW TO USE

1.  **PULL Before You Work**: Before starting an editing session, press **`Ctrl+Alt+R`** to run the "Pull from Server" task. This ensures your local copy is up-to-date with any changes made on the server (like log files).

2.  **Edit and Auto-PUSH**: Open any file and start working. When you save (`Ctrl+S`), the `runonsave` extension will automatically sync your changes to the server.

-----

## 🧪 TESTING YOUR SETUP

You can test the connection by creating a file on the server that updates every second.

1.  SSH into your server: `ssh gpulab`
2.  Navigate to your remote project directory.
3.  Run this one-liner. It will create a `test_log.txt` file and add a new line to it every second.
    ```bash
    while true; do echo "Log entry at $(date)" >> test_log.txt; sleep 1; done
    ```
4.  In VS Code, press **`Ctrl+Alt+R`**. The `test_log.txt` file should appear locally. Open it to see the contents.
5.  Press **`Ctrl+Alt+R`** again a few seconds later. The file will update with the new lines.
6.  When you're done, go to your server terminal and press **`Ctrl+C`** to stop the script.