# Guide: Display current Working Directory with GIT status in Waybar for Omarchy + Ghostty

A guide on how to configure Waybar to display the current working directory (PWD) of your active Ghostty window. This setup also includes Git integration, showing branch and status, it is optional to use event-driven updates via socket signals.

## Prerequisites

- This tutorial is written for Omarchy users. If you are not running omarchy, you will need to find a workaround solution for omarchy-cmd-terminal-cwd.
- Have Python installed

## Step 1: Configure Ghostty

To ensure that each new terminal window opens as a new instance (which is necessary for `omarchy-cmd-terminal-cwd` to work correctly with multiple terminals), you need to modify your Ghostty configuration.

1.  Open your Ghostty configuration file located at `~/.config/ghostty/config`.
2.  Set the `window-inherit-working-directory` option to `false`:

    ```
    window-inherit-working-directory = false
    ```

## Step 2: Rebind Your Terminal Shortcut

You need to modify your Hyprland keybinding to launch Ghostty directly.

1.  Open your Hyprland bindings configuration file (e.g., `~/.config/hypr/bindings.conf`).
2.  Unbind your existing terminal keybinding and rebind it to use `uwsm-app -- ghostty --working-directory="$(omarchy-cmd-terminal-cwd)"`:

    ```
    unbind = SUPER, RETURN
    bind = SUPER, RETURN, exec, uwsm-app -- ghostty --working-directory="$(omarchy-cmd-terminal-cwd)"
    ```

    Replace `SUPER, RETURN` with your preferred key combination for opening a terminal.

## Step 3: Create the Waybar Script

Create the script that Waybar will execute to get the PWD and Git information.

1.  Create a new file named `waybar_pwd.sh` in your Waybar scripts directory (e.g., `~/.config/waybar/scripts/`).
2.  Make the script executable: `chmod +x ~/.config/waybar/scripts/waybar_pwd.sh`
3.  Add the following content to the script:

    ```bash
    #!/usr/bin/env bash

    # Enable strict mode: exit on error, treat unset variables as errors, and propagate pipeline errors
    set -euo pipefail

    # Get our active window information
    json=$(hyprctl activewindow -j 2>/dev/null || echo '{}')
    class=$(jq -r '.initialClass // empty' <<< "$json")
    pid=$(jq -r '.pid // empty' <<< "$json")

    # If no active window or the window is not a terminal, display the date and time
    if [[ -z "$pid" || ( "$class" != "com.mitchellh.ghostty" && "$class" != "Ghostty" ) ]]; then
        DAY=$(date +%A)
        TIME=$(date +%H:%M:%S)
        echo "{"text": "$DAY - $TIME", "class": "noterminal"}"
        exit 0
    fi

    # We are in a terminal, get the current working directory
    cwd=$(omarchy-cmd-terminal-cwd)
    FINALBRANCH=""

    if [[ -n "$cwd" && -d "$cwd" ]]; then
        display=${cwd/#$HOME/\~}

        if git -C "$cwd" rev-parse --is-inside-work-tree >/dev/null 2>&1; then
            # 1. Get Branch Name
            BRANCH=$(git -C "$cwd" branch --show-current 2>/dev/null || git -C "$cwd" rev-parse --short HEAD 2>/dev/null || echo "")
            
            if [[ -z "$BRANCH" ]]; then
              # Detached, .. tag name? otherwise get the SHA
              BRANCH=$(git -C "$cwd" describe --tags --exact-match 2>/dev/null \
                        || git -C "$cwd" rev-parse --short HEAD 2>/dev/null)

              # Wrap in parens to show we are detached
              BRANCH="($BRANCH)"
            fi
            
            # 2. Get Git Status Indicators
            # --porcelain is stable for scripting.
            # Column 1 = Staged, Column 2 = Unstaged/Modified
            STATUS=$(git -C "$cwd" status --porcelain 2>/dev/null)

            STAGED=$(echo "$STATUS" | grep -c "^[MADR]" || true)
            MODIFIED=$(echo "$STATUS" | grep -c ".[MADR]" || true)
            UNTRACKED=$(echo "$STATUS" | grep -c "??" || true)

            GIT_SYMBOLS=""
            # Add a space before symbols if any exist
            if [[ $STAGED -gt 0 || $MODIFIED -gt 0 || $UNTRACKED -gt 0 ]]; then
                GIT_SYMBOLS+=" "

                [[ $STAGED -gt 0 ]] && GIT_SYMBOLS+="+"
                [[ $MODIFIED -gt 0 ]] && GIT_SYMBOLS+="!"
                [[ $UNTRACKED -gt 0 ]] && GIT_SYMBOLS+="?"
            fi

            if [ -n "$BRANCH" ]; then
                FINALBRANCH=" îœ¥ $BRANCH$GIT_SYMBOLS"
            fi
        fi

        echo "{"text": "$display<span color='#9a9a9a'>$FINALBRANCH</span>", "tooltip": "$cwd", "class": "terminal"}"
    else
        DAY=$(date +%A)
        TIME=$(date +%H:%M:%S)
        echo "{"text": "$DAY - $TIME", "class": "noterminal"}"
    fi
    ```

## Step 4: Configure Waybar

Now, we'll add a custom module to the Waybar configuration to display PWD.

1.  Open your Waybar configuration file (e.g., `~/.config/waybar/config.jsonc`).
2.  Add the following custom module:

    ```json
    "custom/pwd": {
        "format": "{}",
        "exec": "~/.config/waybar/scripts/waybar_pwd.sh",
        "interval": 1,
        "return-type": "json",
        "signal": 13
    },
    ```

3.  Enable the module by adding `"custom/pwd"` to your desired module location (e.g., `modules-center`):

    ```json
    "modules-center": ["custom/pwd", "..."],
    ```

## Step 5: Enable the Module

Restart Waybar to apply the changes (e.g., `hyprctl reload` or restart Hyprland).

## Step 6: Create Socket Listener and Configure Autostart

To enable event-driven updates, create a Python script that listens to Hyprland's socket and signals Waybar on window changes.

1.  Create a new file named `watcher.py` in your Waybar scripts directory (e.g., `~/.config/waybar/scripts/watcher.py`).
2.  Add the following content to the script:

    ```python
    import socket
    import os
    import subprocess

    def refresh_waybar_module(signal_num):
        # Sends SIGRTMIN + signal_num to all Waybar instances
        subprocess.run(["pkill", f"-RTMIN+{signal_num}", "waybar"])

    # Get the socket path
    signature = os.environ.get("HYPRLAND_INSTANCE_SIGNATURE")
    runtime_dir = os.environ.get("XDG_RUNTIME_DIR")
    socket_path = f"{runtime_dir}/hypr/{signature}/.socket2.sock"

    client = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
    client.connect(socket_path)

    while True:
        data = client.recv(1024).decode("utf-8")
        for line in data.split("\n"):
            if "activewindowv2" in line:
                # refresh_waybar_module with signal
                refresh_waybar_module(13)
    ```

3.  Open your Hyprland configuration file (e.g., `~/.config/hypr/hyprland.conf`).
4.  Add the following line to the configuration to start the script automatically on Hyprland startup:

    ```
    exec-once = python ~/.config/waybar/scripts/watcher.py
    ```

5.  Restart Hyprland to apply the changes and start the watcher script.

## End

Your Waybar should now display the current working directory of your Ghostty terminal with Git status, updating both on a 1-second interval (for clock simulation) and immediately on window changes via socket signals. When you are not in a terminal, it will fall back to showing the current day and time.
