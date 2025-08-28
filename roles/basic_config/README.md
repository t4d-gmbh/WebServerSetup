### **Basic Configuration** ⚙️

This role manages **foundational server configurations**, primarily focusing on **swap space management** and **kernel parameter tuning** for optimal performance [6-8]. It ensures that your system has adequate swap space and that memory management settings are adjusted according to best practices for server environments.

#### Actions Performed

*   **Checks for existing swap configuration**: Verifies if any swap space is already active on the system [9, 10].
*   **Ensures adequate disk space**: Checks available disk space on the root partition before attempting to create a swap file [11].
*   **Creates a swap file**: If `swapfile_size` is provided and the swap file does not exist, it creates a new swap file using the `dd` command to prevent "holes" in the file, which is a community-recommended method for swap usage [conversation history, 41]. The DigitalOcean tutorial initially suggested `fallocate` for this [12].
*   **Sets secure file permissions**: Configures the swap file with **strict permissions (`0600`)**, allowing only the root user to read and write, thereby preventing significant security implications [13, 14].
*   **Marks the file as swap space**: Initializes the newly created file as Linux swap space using `mkswap` [14].
*   **Activates the swap file**: Enables the swap file for immediate use by the system using `swapon` [15].
*   **Makes swap permanent**: Adds an entry to `/etc/fstab` to ensure the swap file is activated automatically on system reboots, backing up the original `fstab` file first [16].
*   **Adjusts `vm.swappiness`**: Configures how aggressively the system swaps data out of RAM to the swap space. For servers, a **lower value (e.g., 10)** is often preferred to keep more data in faster RAM and reduce disk I/O, as interactions with swap are slower than RAM [7, 17, 18].
*   **Adjusts `vm.vfs_cache_pressure`**: Controls how much the kernel prioritizes caching filesystem metadata (like *inode* and *dentry* information). A **lower value (e.g., 50)** helps retain this frequently accessed and costly-to-lookup data in the cache, improving overall filesystem performance [8, 19].

#### Variables

The role can be configured using the following variables:

*   **`swapfile_size`**: (Optional) Specifies the desired size of the swap file, e.g., `"1G"` or `"2048M"`.
    *   **If this variable is `none` or an empty string, no swap file will be created or configured** [conversation history].
    *   Generally, an amount equal to or double the system's RAM is a good starting point, though anything over 4G may be unnecessary if only used as a RAM fallback [20].
*   **`swapfile_path`**: (Default: `/swapfile`) The absolute path where the swap file will be created [conversation history, 29].
*   **`swappiness_value`**: (Default: `10`) A value between 0 and 100 that determines how often the system swaps data to disk. Lower values (closer to 0) are generally recommended for servers [7, 17, 18].
*   **`vfs_cache_pressure_value`**: (Default: `50`) A value between 0 and 100 that influences how much the system prioritizes caching filesystem metadata. A lower value (e.g., 50) is recommended for better performance [8, 19].

#### Example Usage

To use this role in your playbook, include it in your roles list and define the necessary variables. For instance:

```yaml
---
- name: Apply basic server configurations
  hosts: all
  become: yes
  roles:
    - role: t4d.WebServerSetup.basic_config
      vars:
        swapfile_size: "2G"
        swappiness_value: 10
        vfs_cache_pressure_value: 50
```
This example would configure a 2GB swap file and set the `swappiness` and `vfs_cache_pressure` to their recommended server values.
