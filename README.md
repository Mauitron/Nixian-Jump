# Nixian-Jump Documentation
![image](https://github.com/user-attachments/assets/1c486def-7816-4242-9205-44467b6ab6fc)
If you like this tool, why not [Buy Me a Coffee](https://buymeacoffee.com/charon0)?
## Overview
`Nixian-jump` is a navigation tool for NixOS configuration files that provides
a hierarchical menu interface for quickly jumping to different sections of your
configuration. It was designed for the Helix editor (hence the 'hx' down below),
but should work with any other editor that can use line number navigation.

## Requirements

### System Components
- **NixOS Configuration**: Configuration file at `/etc/nixos/configuration.nix`
- **Shell Tools**:
  - `bash` (scripting)
  - `grep`, `sed`, `awk`, `column` (text parsing)
- **Wayland Components**:
  - `wofi` (menu interface)
- **Input Simulation**:
  - `ydotool` (keyboard input simulation)
  - `ydotoold` (daemon service)

### Package Installation
Add the following to your `configuration.nix`:

```nix
environment.systemPackages = with pkgs; [
  wofi
  ydotool
];
```

Run `sudo nixos-rebuild switch` after making changes.

## Implementation Guide

### 1. Script Installation
Add this code to your NixOS configuration under the environment.systemPackages section:

```nix
environment.systemPackages = with pkgs; [
  (pkgs.writeScriptBin "hx-jump" ''
    #!/usr/bin/env bash

    # Ensure ydotoold is running
    if ! pgrep -x "ydotoold" > /dev/null; then
      echo "ydotoold is not running. Starting it..."
      systemctl --user start ydotool
      sleep 1
    fi

    CONFIG_FILE="/etc/nixos/configuration.nix"

    # Function to create menu items based on parent category
    show_menu() {
      local prompt_text="$1"

      # First, get main categories
      {
        grep -n "^#[0-9]\+\[\(.*\)\]" "$CONFIG_FILE" | \
        sed 's/:#\([0-9]*\)\[\(.*\)\]/|\1|\2/' | \
        awk -F'|' '{printf "%s|%s\n", $1, $3}'

        # Get subcategories (those with >)
        grep -n "^#[0-9]\+>\[\(.*\)\]" "$CONFIG_FILE" | \
        sed 's/:#\([0-9]*\)>\[\(.*\)\]/|\1|    ┗━ \2/' | \
        awk -F'|' '{printf "%s|%s\n", $1, $3}'

      } | sort -n | \
        column -t -s'|' | \
        wofi --dmenu \
             --prompt "$prompt_text" \
             --width 400 \
             --height 500 \
             --cache-file /dev/null \
   #         --style /etc/xdg/wofi/minimize-style.css \ 
             --hide-scroll \
             --insensitive \
             --normal-window
    }

    # Show the hierarchical menu
    selection=$(show_menu "Jump to section:")

    if [ -n "$selection" ]; then
      # Extract the line number from the selection
      line_number=$(echo "$selection" | awk '{print $1}')

      if [ -n "$line_number" ]; then
        ${pkgs.ydotool}/bin/ydotool type "$line_number"
        ${pkgs.ydotool}/bin/ydotool type "G" #In ydotool,"U" is "G" for dvorak
      fi
    fi
  '')
];
```

### 2. System Configuration
After adding the script:

#1. Rebuild your system:
```bash
sudo nixos-rebuild switch
```

#2. Configure ydotool daemon service:
```nix
 systemd.services.ydotool = {
    description = "ydotool daemon";
    wantedBy = [ "multi-user.target" ];
    serviceConfig = {
      ExecStart = "${pkgs.ydotool}/bin/ydotoold";
      Restart = "always";
    };
  };
```

## Syntax Explanation

The script uses a special syntax in your configuration file to mark sections:

### Main Categories
```
#1[Category Name]

```
- `#` marks the start of a category
- `1` the category number
- Text inside `[]` is the category name

#### Subcategories
```
#1>[Subcategory Name]

```
- Similar to main categories but includes `>`
- The `>` indicates this is a subcategory
- Displayed with a tree branch (`┗━`) in the menu

### Examples
```nix
#1[System Configuration]
# ... configuration items ...

#1>[Boot Options]
# ... boot-related configuration ...

#1>[Network Settings]
# ... network-related configuration ...
```
In the example, [Boot Options] and [Network Settings] are subcategories of [System Configuration]

## Usage
1. Ensure `ydotoold` is running (`systemctl --user start ydotool`)
2. Run `hx-jump` from your terminal to see if it works
3. implement a way to run it from you editor
4. Select a category from the wofi menu
5. The script will automatically jump to the selected section in your configuration file
6. Tip: if you are using Helix, I recommend adding something like this to your config file

![image](https://github.com/user-attachments/assets/32ff559d-56fa-4eeb-ad4c-1484e9550eed)


The script combines these numbered sections with wofi's menu interface to create a hierarchical navigation system for your NixOS configuration.
