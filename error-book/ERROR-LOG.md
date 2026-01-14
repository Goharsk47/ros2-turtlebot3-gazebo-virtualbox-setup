# Error Book - ROS 2 Humble & TurtleBot3 Gazebo Setup

This document records all errors, issues, and their fixes encountered during the setup of Gazebo 11 and TurtleBot3 on Ubuntu 22.04 with ROS 2 Humble in a VirtualBox environment.

---

## Error 1: Gazebo OSRF Repository & GPG Key Issues

### Context
- OS: Ubuntu 22.04 (Jammy) in VirtualBox
- Task: Adding OSRF Gazebo repository and GPG key
- ROS Version: ROS 2 Humble

### Error Messages
```
curl: (6) Could not resolve host: gazebo.gpg--output
E: Malformed entry 1 in list file /etc/apt/sources.list.d/gazebo-stable.list (Component)
E: The list of sources could not be read.
Permission denied: Cannot write to /etc/apt/sources.list.d/
```

### Root Cause
1. **Missing space in curl command**: URL and `--output` flag were concatenated (`gazebo.gpg--output` instead of `gazebo.gpg --output`)
2. **Quote mismatch in echo command**: Using `sh -c` with mismatched quotes caused the terminal to hang
3. **Corrupted .list file**: Accidentally typed `exit` and `null` into the `.list` file during troubleshooting

### Solution
```bash
# Clean up corrupted files
sudo rm /etc/apt/sources.list.d/gazebo-stable.list

# Correct curl command with space between URL and flag
sudo curl -sSL https://packages.osrfoundation.org/gazebo.gpg \
  -o /usr/share/keyrings/pkgs-osrf-archive-keyring.gpg

# Correct echo with proper pipe to sudo tee
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/pkgs-osrf-archive-keyring.gpg] http://packages.osrfoundation.org/gazebo/ubuntu-stable $(lsb_release -cs) main" \
  | sudo tee /etc/apt/sources.list.d/gazebo-stable.list > /dev/null

# Update package lists
sudo apt update
```

### Prevention
- Always check command syntax carefully, especially with pipes and redirects
- Use `cat` to verify file contents after writing: `sudo cat /etc/apt/sources.list.d/gazebo-stable.list`
- Never use `sh -c` with echo for writing to protected files; use pipe to `sudo tee` instead

---

## Error 2: Dependency Conflicts (Gazebo 11 vs Gazebo Sim)

### Context
- Attempting to install `ros-humble-turtlebot3-gazebo`
- System had conflicting Gazebo packages

### Error Messages
```
Unmet dependencies:
E: Unmet dependencies. Try using -f.
The following packages have unmet dependencies:
  gazebo11 : Breaks: gz-tools2
  gz-tools2 : Breaks: gazebo11
```

### Root Cause
Newer `gz-tools2` (from Gazebo Sim/Ignition) was pre-installed or installed as a dependency, conflicting with Gazebo Classic 11 which TurtleBot3 requires.

### Solution
```bash
# Remove the conflicting package
sudo apt remove -y gz-tools2

# Install Gazebo 11 and dependencies
sudo apt install -y gazebo11 libgazebo11-dev

# Install TurtleBot3 packages
sudo apt install -y ros-humble-turtlebot3-gazebo
sudo apt install -y ros-humble-turtlebot3-simulations

# Install ROS-Gazebo bridge
sudo apt install -y ros-humble-gazebo-ros-pkgs
```

### Prevention
- Always check for conflicting packages before installing new ones: `apt-cache depends gazebo11`
- Pin Gazebo 11 to prevent auto-updates to Gazebo Sim: `sudo apt-mark hold gazebo11`
- Check ROS package documentation for compatibility info

---

## Error 3: Service /spawn_entity Unavailable

### Context
- Gazebo world loaded but TurtleBot3 robot did not appear
- Spawn script timed out waiting for service

### Error Messages
```
[spawn_entity-2] Service /spawn_entity unavailable after 10.0 seconds.
[ERROR] [spawn_entity-2]: Failed to connect to service /spawn_entity
```

### Root Cause
1. **Missing ROS-Gazebo bridge packages**: The ROS 2 plugins that communicate between ROS and Gazebo were not installed
2. **VirtualBox graphics limitations**: 3D acceleration issues prevented Gazebo plugins from loading correctly
3. **Missing environment variables**: Software rendering mode not enabled for VirtualBox

### Solution
```bash
# Install ROS-Gazebo bridge (if not already done)
sudo apt install -y ros-humble-gazebo-ros-pkgs

# Set environment variables for VirtualBox
export LIBGL_ALWAYS_SOFTWARE=1
export TURTLEBOT3_MODEL=burger

# Source ROS 2 setup
source /opt/ros/humble/setup.bash

# Launch Gazebo with proper settings
ros2 launch turtlebot3_gazebo turtlebot3_world.launch.py
```

### Prevention
- Always check `/etc/apt/sources.list.d/` is clean before installing packages
- Set environment variables in `~/.bashrc` to persist across sessions
- Test Gazebo independently before testing TurtleBot3: `gazebo`
- Check plugin logs: `cat ~/.gazebo/server-11999/gzserver.log`

---

## Notes for Future Troubleshooting

1. **VirtualBox 3D Acceleration**: 
   - Can be disabled in VM settings to improve stability
   - Always use software rendering: `LIBGL_ALWAYS_SOFTWARE=1`

2. **Package Management**:
   - Use `apt-cache` to check dependencies before installing
   - Keep `/etc/apt/sources.list.d/` clean and well-organized

3. **Gazebo/ROS Debugging**:
   - Check logs: `~/.gazebo/server-*/gzserver.log`
   - Use `gazebo --verbose` for detailed output
   - Test services with: `ros2 service list`

4. **Performance**:
   - Allocate sufficient VRAM to VirtualBox VM
   - Use `burger` model (lighter) instead of `waffle` if system struggles
