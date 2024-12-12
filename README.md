# Nixian-Jump Documentation!
![output_optimized](https://github.com/user-attachments/assets/90d34450-7416-4174-8a1d-d575c9b5de32)



If you like this tool, why not [Buy Me a Coffee](https://buymeacoffee.com/charon0)☕
## Overview
# Nixian Jump - NixOS Configuration Navigator

Nixian Jump is a navigation tool for NixOS configuration files that provides a hierarchical menu interface for quickly jumping to different sections of your configuration. It works with any editor that supports line number navigation, with specific optimizations for Helix.

## Requirements

### System Components
- NixOS Configuration file at `/etc/nixos/configuration.nix`
- Required packages:
  - rofi (menu interface)
  - ydotool (keyboard input simulation)
  - ydotoold (daemon service)

## Installation Guide

### 1. Add Required Packages

Add these packages to your `configuration.nix`:

```nix
environment.systemPackages = with pkgs; [
  rofi-wayland  # Use rofi-wayland for Wayland compatibility
  ydotool
];
```

### 2. Configure the ydotool Service

Add this service configuration to your `configuration.nix`:

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

### 3. Add the Nixian Jump Script

Add this script configuration to your `configuration.nix`:

```nix
environment.systemPackages = with pkgs; [
  (writeScriptBin "hx-jump" ''
    #!/bin/sh
    
    if ! pgrep -x "ydotoold" > /dev/null; then
      echo "ydotoold is not running. Starting it..."
      systemctl --user start ydotool
      sleep 1
    fi

    CONFIG_FILE="/etc/nixos/configuration.nix"

    show_menu() {
      local prompt_text="$1"
      
      {
        echo "# Main Sections"
        grep -n "^#[0-9]\+\[\(.*\)\]" "$CONFIG_FILE" | \
        sed 's/:#\([0-9]*\)\[\(.*\)\]/|\1|\2/' | \
        awk -F'|' '{printf "%s|%s\n", $1, $3}'
        
        echo "# Sub Sections"
        grep -n "^#[0-9]\+>\[\(.*\)\]" "$CONFIG_FILE" | \
        sed 's/:#\([0-9]*\)>\[\(.*\)\]/|\1|    ┗━ \2/' | \
        awk -F'|' '{printf "%s|%s\n", $1, $3}'
        
      } | sort -n | \
        column -t -s'|' | \
        rofi -dmenu \
             -p "$prompt_text" \
             -theme-str '
                window {
                    width: 800px;
                    height: 600px;
                    location: center;
                    anchor: center;
                    transparency: "real";
                    background-color: #2E3440;
                    border: 3px solid;
                    border-color: #4C566A;
                    border-radius: 12px;
                }
                mainbox {
                    background-color: transparent;
                    children: [inputbar, listview];
                }
                inputbar {
                    padding: 0px;
                    margin: 0px 0px 20px 0px;
                    background-color: #3B4252;
                    text-color: #ECEFF4;
                    border-radius: 8px;
                    border: 1px solid;
                    border-color: #4C566A;
                    children: [prompt, textbox-prompt-colon, entry];
                }
                prompt {
                    enabled: true;
                    padding: 12px;
                    background-color: transparent;
                    text-color: inherit;
                }
                textbox-prompt-colon {
                    expand: false;
                    str: "";
                    padding: 12px;
                    text-color: inherit;
                }
                entry {
                    padding: 12px;
                    background-color: transparent;
                    text-color: inherit;
                    placeholder: "Search sections...";
                    placeholder-color: #4C566A;
                }
                listview {
                    columns: 1;
                    lines: 10;
                    scrollbar: true;
                    padding: 10px;
                    background-color: transparent;
                    border: 0px;
                }
                scrollbar {
                    width: 4px;
                    padding: 0;
                    handle-width: 8px;
                    border: 0;
                    handle-color: #4C566A;
                }
                element {
                    padding: 12px;
                    background-color: transparent;
                    text-color: #ECEFF4;
                    border-radius: 8px;
                }
                element normal.normal {
                    background-color: transparent;
                    text-color: #ECEFF4;
                }
                element selected.normal {
                    background-color: #3B4252;
                    text-color: #88C0D0;
                    border: 1px solid;
                    border-color: #4C566A;
                }
                element-text {
                    background-color: transparent;
                    text-color: inherit;
                    highlight: bold #88C0D0;
                }
             ' \
             -matching fuzzy \
             -i \
             -no-custom \
             -hover-select \
             -me-select-entry "" \
             -me-accept-entry "MousePrimary"
    }

    if [ ! -r "$CONFIG_FILE" ]; then
      echo "Error: Cannot read $CONFIG_FILE"
      exit 1
    fi

    selection=$(show_menu "Jump to section")

    if [ -n "$selection" ]; then
      line_number=$(echo "$selection" | awk '{print $1}')
      
      if [ -n "$line_number" ]; then
        ${pkgs.ydotool}/bin/ydotool type "$line_number"
        ${pkgs.ydotool}/bin/ydotool type "G" # the navigation key in helix
      fi
    fi
  '')
];
```

### 4. Apply Changes

After adding all components, rebuild your system:

```bash
sudo nixos-rebuild switch
```

## Configuration File Syntax

Nixian Jump uses a special syntax in your configuration file to mark sections:

### Main Categories
```nix
#1[Category Name]
```
- `#` marks the start of a category
- `1` is the category number
- Text inside `[]` is the category name

### Subcategories
```nix
#1>[Subcategory Name]
```
- Similar to main categories but includes `>`
- The `>` indicates this is a subcategory
- Displayed with a tree branch (┗━) in the menu

### Example Structure
```nix
#1[System Configuration]
# ... configuration items ...

#1>[Boot Options]
# ... boot-related configuration ...

#1>[Network Settings]
# ... network-related configuration ...
```

In the example, [Boot Options] and [Network Settings] are subcategories of [System Configuration]

## Editor Integration

### Helix Integration

If you are using Helix, add this to your Helix configuration (`config.toml`):

```toml
[keys.normal.space]
u = [":sh hx-jump"]
```

This allows you to trigger Nixian Jump with `Space + u` in normal mode.

## Usage

1. Ensure `ydotoold` is running:
   ```bash
   systemctl --user start ydotool
   ```
2. Run `hx-jump` from your terminal or trigger it from your editor
3. Use the rofi menu to select a section:
   - Type to search (fuzzy-find enabled)
   - Use arrow keys, tab or mouse to navigate the options
   - Press Enter or click to select
4. The script will automatically jump to the selected section in your configuration file

## Features

- Hierarchical navigation system
- Fuzzy search functionality
- Beautiful Nord-themed interface
- Mouse and keyboard navigation
- Section categorization with visual hierarchy
- Instant jumping to any config section
