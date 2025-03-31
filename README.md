# GaiaNet Multi-Instance Installer Script & User Guide

This repository provides a Bash script that automates the installation and management of multiple GaiaNet nodes on your VPS. With this tool you can:

- **Select a Model:** Choose from 4 pre-defined model configurations or provide your own custom URL.
- **Automatic Instance Management:** Automatically detect the next available instance number and assign unique ports.
- **Installation & Startup:** Install, initialize, configure, and start each GaiaNet node.
- **Logging:** Save Node ID and Device ID for each instance in a dedicated log folder.
- **Helpful Commands:** Easily start, stop, and view node information.

---

## Prerequisites

- Ubuntu 20.04/22.04 (or similar)
- Basic utilities such as `curl` and `bash`
- Sufficient system resources to run multiple GaiaNet nodes

---

## Script: `install_gaia_instances.sh`

Below is the complete script. Save it as `install_gaia_instances.sh`, make it executable, and run it to install your nodes.

```bash
#!/bin/bash
# GaiaNet multi-instance installer with model selection and automatic instance/port management

set -e

echo "Select the model config you want to use:"
echo "1) Qwen 1.5 0.5B Chat"
echo "   https://raw.githubusercontent.com/GaiaNet-AI/node-configs/main/qwen-1.5-0.5b-chat/config.json"
echo "2) Qwen2 0.5B Instruct"
echo "   https://raw.githubusercontent.com/GaiaNet-AI/node-configs/main/qwen2-0.5b-instruct/config.json"
echo "3) Qwen 2.5 Coder 0.5B Instruct"
echo "   https://raw.githubusercontent.com/GaiaNet-AI/node-configs/main/qwen-2.5-coder-0.5b-instruct/config.json"
echo "4) LLaMA 3.2 3B Instruct"
echo "   https://raw.githubusercontent.com/GaiaNet-AI/node-configs/main/llama-3.2-3b-instruct/config.json"
echo "5) Custom URL"

read -p "Enter your choice [1-5]: " model_choice

# Choose config link based on input
case "$model_choice" in
  1)
    config_url="https://raw.githubusercontent.com/GaiaNet-AI/node-configs/main/qwen-1.5-0.5b-chat/config.json"
    ;;
  2)
    config_url="https://raw.githubusercontent.com/GaiaNet-AI/node-configs/main/qwen2-0.5b-instruct/config.json"
    ;;
  3)
    config_url="https://raw.githubusercontent.com/GaiaNet-AI/node-configs/main/qwen-2.5-coder-0.5b-instruct/config.json"
    ;;
  4)
    config_url="https://raw.githubusercontent.com/GaiaNet-AI/node-configs/main/llama-3.2-3b-instruct/config.json"
    ;;
  5)
    read -p "Paste your full config URL: " config_url
    ;;
  *)
    echo "❌ Invalid option. Exiting."
    exit 1
    ;;
esac

echo "✅ Using config: $config_url"
echo

read -p "Enter the number of GaiaNet instances to create: " num_instances

# Auto-detect the next available instance number based on existing folders
existing=$(ls -d "$HOME"/gaia-node-* 2>/dev/null | sed 's/.*gaia-node-//' | sort -n | tail -1)
if [ -z "$existing" ]; then
    instance_start=101
else
    instance_start=$((existing + 1))
fi

# Set the starting port based on instance number (e.g., instance 101 -> port 8101)
port_start=$((8100 + instance_start))
info_dir="$HOME/gaia-node-info"
mkdir -p "$info_dir"

for (( i=0; i<num_instances; i++ ))
do
    instance_number=$(( instance_start + i ))
    instance_dir="$HOME/gaia-node-${instance_number}"
    port=$(( port_start + i ))

    echo "--------------------------------------------------"
    echo "Installing GaiaNet instance: $instance_number (Port: $port)"

    mkdir -p "$instance_dir"

    # Run install script for this instance
    curl -sSfL 'https://github.com/GaiaNet-AI/gaianet-node/releases/latest/download/install.sh' | bash -s -- --base "$instance_dir"

    # Add instance bin path to current PATH
    export PATH="$instance_dir/bin:$PATH"

    # Initialize the node with the selected model config
    gaianet init --base "$instance_dir" --config "$config_url"

    # Update port configuration for the instance
    gaianet config --base "$instance_dir" --port "$port"

    # Run initialization again (if required)
    gaianet init --base "$instance_dir"

    # Start the GaiaNet node
    gaianet start --base "$instance_dir"

    # Save Node ID and Device ID to a log file
    info_file="${info_dir}/node_info_${instance_number}.txt"
    gaianet info --base "$instance_dir" > "$info_file"

    echo "✅ Instance $instance_number started. Info saved to $info_file"
done

echo "--------------------------------------------------"
echo "🎉 All $num_instances instances installed with model config:"
echo "$config_url"
```
## 📖 How to Use the Script

### 1. Save the Script

Create a new file:

    nano install_gaia_instances.sh

Paste the full script into the file, then save and exit:

- Press `Ctrl + O`, then `Enter` to save  
- Press `Ctrl + X` to exit

---

### 2. Make the Script Executable

    chmod +x install_gaia_instances.sh

---

### 3. Run the Script

    ./install_gaia_instances.sh

You will be prompted to:

- Select a model configuration (1–4) or input a custom config URL (option 5)
- Enter the number of GaiaNet instances to install

The script will:

- Automatically detect the next available instance number (e.g., `gaia-node-106`)
- Assign a unique port to each instance (e.g., 8106, 8107, ...)
- Install, initialize, and start each node
- Save Node ID and Device ID to log files

---

### 4. File Locations

- **Node directories:**  
  Each instance is created in its own folder:
  
      ~/gaia-node-101
      ~/gaia-node-102
      ...

- **Node info logs:**  
  Node ID & Device ID are stored in:

      ~/gaia-node-info/node_info_101.txt
      ~/gaia-node-info/node_info_102.txt
      ...

---

## 🧰 Helpful Commands

### ✅ Start All Nodes

    for d in ~/gaia-node-*; do [ -x "$d/bin/gaianet" ] && echo "Starting $d" && export PATH="$d/bin:$PATH" && gaianet start --base "$d"; done

---

### 🛑 Stop All Nodes

    for d in ~/gaia-node-*; do [ -x "$d/bin/gaianet" ] && echo "Stopping $d" && export PATH="$d/bin:$PATH" && gaianet stop --base "$d"; done

---

### 📋 View Node IDs and Device IDs

    grep -H -E "Node ID|Device ID" ~/gaia-node-info/node_info_*.txt

---

## ℹ️ Additional Information

- **Instance Numbering:**  
  The script auto-detects the next available instance number by scanning existing `gaia-node-*` directories. If no previous instances exist, it starts from `101`.

- **Port Assignment:**  
  Ports are auto-assigned based on the instance number:
  
      Port = 8100 + instance_number

- **Customization Ideas:**  
  - Run nodes in the background using `tmux` or `screen`
  - Integrate with `systemd` for auto-start on reboot
  - Add logging to capture runtime output per instance

- **Logs & Monitoring:**  
  Check logs in `~/gaia-node-info/` to view Node ID and Device ID.

- **Support & Contributions:**  
  Feel free to submit issues or pull requests if you want to improve the script or add new features.

---

Happy Node Running 🚀
