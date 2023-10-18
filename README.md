# get-system-info-in-linux
This repo is more like a note for me but it will retrieve Network (IP, interfaces and MAC), total vCPU / Memory and Disks name and size from each Linux host using ansible playbook

## How to run and prepare this play book to get system info
### Playbook
```yaml
---
- name: Get System Information for Network Interfaces, MacAddress, Total vCPU/Memory/IpAddress and Disk info
  hosts: my_hosts
  gather_facts: no

  tasks:
    - name: Execute shell script to get network interfaces, MAC addresses, and IP addresses
      shell: |
        ip a | egrep "$filter_pattern" | grep -E '^[0-9]+: [a-zA-Z0-9]+' | awk '{print $2}' | sed 's/://' | sort -n | while read -r iface; do
            mac_address="$(ip link show "$iface" | awk '/ether/ {print $2}')"
            ip_address="$(ip a show "$iface" | awk '/inet / {print $2}' | cut -d'/' -f1 | head -n 1)"
            if [ -n "$ip_address" ]; then
                echo "{interface: $iface, mac_address: $mac_address, ip_address: $ip_address}"
            else
                echo "{interface: $iface, mac_address: $mac_address}"
            fi
        done              
      register: network_info
      environment:
        filter_pattern: "{{ filter_pattern }}"  # Pass the filter pattern as an environment variable

    - name: Get Total vCPUs
      shell: nproc
      register: vcpu_output

    - name: Get Total Memory in GB
      shell: awk '/MemTotal/ {print $2 / 1024 / 1024}' /proc/meminfo
      register: memory_output

    - name: Execute lsblk command to get disk information
      shell: lsblk --list -o NAME,SIZE
      register: disk_info

    - name: Set and save disk info 
      set_fact:
        disk_sizes: "{{ disk_info.stdout_lines[1:] }}"
      #run_once: true

    - name: "Filter BM disk sizes and name"
      set_fact:
        filtered_disk_sizes: "{{ filtered_disk_sizes | default([]) + [{'name': item.split()[0], 'size': item.split()[1]}] }}"
      loop: "{{ disk_sizes }}"
      when: item.split()[0]  | regex_search('^(sd[a-f]|nvme[0-9]+n[0-9]+)$')

    - name: Display Network Interfaces and MAC Addresses as JSON
      debug:
        msg: "{{ network_info.stdout_lines }}"

    - name: Display Total vCPUs and Memory
      debug:
        msg: "Total vCPUs: {{ vcpu_output.stdout | int }}, Total Memory (GB): {{ memory_output.stdout | float }}"        

    - name: "Display Disk Info (Name and sizes)"
      debug:
        msg: "{{ filtered_disk_sizes }}"

    - name: Unset disk_sizes in case of more than one host
      set_fact:
        disk_sizes: ''        

    - name: Unset filtered_disk_sizes in case of more than one host
      set_fact:
        filtered_disk_sizes: ''
...
```
### Inventory
- With SSHLESS 
```shellSession
[my_hosts]
192.168.xx.xx
192.168.xx.xx
```
- With SSHKEY
```shellSession
[my_hosts]
192.168.xx.xx ansible_ssh_user=myuser ansible_ssh_private_key_file=~/home/xxx/.ssh/id_rsa
```
- With SSHKEY
```shellSession
[my_hosts]
192.168.xx.xx ansible_ssh_user=myuser ansible_ssh_password=mypass
```

### Run the play book
You can pass into the argument your interface prefix and hard-coded disk names are `sd[a-f]` and `nvme[....]`
```shellSession
ansible-playbook -i inv.yml get-sys-info.yml --extra-vars "filter_pattern=ens|eno"
```

### Check the Output
```bash
TASK [Display Network Interfaces and MAC Addresses as JSON] **************************************************************************************************************************
ok: [192.xx.yy.12] => {
    "msg": [
        "{interface: eno12399, mac_address: b4:96:91:xx:7a:a4, ip_address: 192.xx.yy.12}",
        "{interface: eno12409, mac_address: b4:96:91:xx:7a:a5}",
        "{interface: eno8303, mac_address: b0:7b:25:yy:bc:dc}",
        "{interface: eno8403, mac_address: b0:7b:25:yy:bc:dd}",
        "{interface: ens2f0, mac_address: b4:96:91:zz:e5:e8}",
        "{interface: ens2f1, mac_address: b4:96:91:zz:e5:e9}"
    ]
}
ok: [192.xx.yy.13] => {
    "msg": [
        "{interface: eno12399, mac_address: b4:96:91:xx:7b:48, ip_address: 192.xx.yy.13}",
        "{interface: eno12409, mac_address: b4:96:91:xx:7b:49}",
        "{interface: eno8303, mac_address: b0:7b:25:yy:b7:30}",
        "{interface: eno8403, mac_address: b0:7b:25:yy:b7:31}",
        "{interface: ens2f0, mac_address: b4:96:91:zz:e6:38}",
        "{interface: ens2f1, mac_address: b4:96:91:zz:e6:39}"
    ]
}
ok: [192.xx.yy.14] => {
    "msg": [
        "{interface: eno12399, mac_address: b4:96:91:xx:7a:94, ip_address: 192.xx.yy.14}",
        "{interface: eno12409, mac_address: b4:96:91:xx:7a:95}",
        "{interface: eno8303, mac_address: b0:7b:25:yy:b3:b6}",
        "{interface: eno8403, mac_address: b0:7b:25:yy:b3:b7}",
        "{interface: ens2f0, mac_address: b4:96:91:zz:e6:4c}",
        "{interface: ens2f1, mac_address: b4:96:91:zz:e6:4d}"
    ]
}
ok: [192.xx.yy.15] => {
    "msg": [
        "{interface: eno12399, mac_address: b4:96:91:xx:7a:c0, ip_address: 192.xx.yy.15}",
        "{interface: eno12409, mac_address: b4:96:91:xx:7a:c1}",
        "{interface: eno8303, mac_address: b0:7b:25:yy:b7:ae}",
        "{interface: eno8403, mac_address: b0:7b:25:yy:b7:af}"
    ]
}
ok: [192.xx.yy.17] => {
    "msg": [
        "{interface: eno12399, mac_address: b4:96:91:xx:7f:ec, ip_address: 192.xx.yy.17}",
        "{interface: eno12409, mac_address: b4:96:91:xx:7f:ed}",
        "{interface: eno8303, mac_address: b0:7b:25:yy:b1:98}",
        "{interface: eno8403, mac_address: b0:7b:25:yy:b1:99}"
    ]
}
ok: [192.xx.yy.18] => {
    "msg": [
        "{interface: eno12399, mac_address: b4:96:91:ec:52:44, ip_address: 192.xx.yy.18}",
        "{interface: eno12409, mac_address: b4:96:91:ec:52:45}",
        "{interface: eno8303, mac_address: d0:8e:79:f7:a3:42}",
        "{interface: eno8403, mac_address: d0:8e:79:f7:a3:43}",
        "{interface: ens2f0, mac_address: 10:70:fd:b6:11:56}",
        "{interface: ens2f1, mac_address: 10:70:fd:b6:11:57}",
        "{interface: ens5f0, mac_address: b4:96:91:ec:68:7c}",
        "{interface: ens5f1, mac_address: b4:96:91:ec:68:7d}",
        "{interface: ens7f0, mac_address: 10:70:fd:b6:0c:f6}",
        "{interface: ens7f1, mac_address: 10:70:fd:b6:0c:f7}"
    ]
}
ok: [192.xx.yy.19] => {
    "msg": [
        "{interface: eno12399, mac_address: b4:96:91:ec:31:ec, ip_address: 192.xx.yy.19}",
        "{interface: eno12409, mac_address: b4:96:91:ec:31:ed}",
        "{interface: eno8303, mac_address: d0:8e:79:yy:66:fc}",
        "{interface: eno8403, mac_address: d0:8e:79:yy:66:fd}",
        "{interface: ens2f0, mac_address: 10:70:fd:cb:1a:56}",
        "{interface: ens2f1, mac_address: 10:70:fd:cb:1a:57}",
        "{interface: ens5f0, mac_address: b4:96:91:ec:65:e0}",
        "{interface: ens5f1, mac_address: b4:96:91:ec:65:e1}",
        "{interface: ens7f0, mac_address: 10:70:fd:cb:1a:66}",
        "{interface: ens7f1, mac_address: 10:70:fd:cb:1a:67}"
    ]
}
ok: [192.xx.yy.20] => {
    "msg": [
        "{interface: eno12399, mac_address: b4:96:91:ec:52:54, ip_address: 192.xx.yy.20}",
        "{interface: eno12409, mac_address: b4:96:91:ec:52:55}",
        "{interface: eno8303, mac_address: d0:8e:79:f6:c9:42}",
        "{interface: eno8403, mac_address: d0:8e:79:f6:c9:43}",
        "{interface: ens2f0, mac_address: 10:70:fd:b6:11:4e}",
        "{interface: ens2f1, mac_address: 10:70:fd:b6:11:4f}",
        "{interface: ens5f0, mac_address: b4:96:91:ec:65:92}",
        "{interface: ens5f1, mac_address: b4:96:91:ec:65:93}",
        "{interface: ens7f0, mac_address: 10:70:fd:b6:10:5e}",
        "{interface: ens7f1, mac_address: 10:70:fd:b6:10:5f}"
    ]
}

TASK [Display Total vCPUs and Memory] ************************************************************************************************************************************************
ok: [192.xx.yy.12] => {
    "msg": "Total vCPUs: 112, Total Memory (GB): 125.09"
}
ok: [192.xx.yy.13] => {
    "msg": "Total vCPUs: 112, Total Memory (GB): 125.09"
}
ok: [192.xx.yy.14] => {
    "msg": "Total vCPUs: 112, Total Memory (GB): 125.09"
}
ok: [192.xx.yy.15] => {
    "msg": "Total vCPUs: 112, Total Memory (GB): 251.091"
}
ok: [192.xx.yy.17] => {
    "msg": "Total vCPUs: 112, Total Memory (GB): 251.096"
}
ok: [192.xx.yy.18] => {
    "msg": "Total vCPUs: 112, Total Memory (GB): 251.087"
}
ok: [192.xx.yy.19] => {
    "msg": "Total vCPUs: 112, Total Memory (GB): 251.087"
}
ok: [192.xx.yy.20] => {
    "msg": "Total vCPUs: 112, Total Memory (GB): 251.087"
}

TASK [Display Disk Info (Name and sizes)] ********************************************************************************************************************************************
ok: [192.xx.yy.12] => {
    "msg": [
        {
            "name": "nvme0n1",
            "size": "894.3G"
        }
    ]
}
ok: [192.xx.yy.13] => {
    "msg": [
        {
            "name": "nvme0n1",
            "size": "894.3G"
        }
    ]
}
ok: [192.xx.yy.14] => {
    "msg": [
        {
            "name": "nvme0n1",
            "size": "894.3G"
        }
    ]
}
ok: [192.xx.yy.15] => {
    "msg": [
        {
            "name": "sda",
            "size": "446.6G"
        },
        {
            "name": "nvme0n1",
            "size": "2.9T"
        },
        {
            "name": "nvme1n1",
            "size": "2.9T"
        }
    ]
}
ok: [192.xx.yy.17] => {
    "msg": [
        {
            "name": "nvme0n1",
            "size": "2.9T"
        },
        {
            "name": "nvme1n1",
            "size": "2.9T"
        }
    ]
}
ok: [192.xx.yy.18] => {
    "msg": [
        {
            "name": "sda",
            "size": "446.6G"
        },
        {
            "name": "sdb",
            "size": "446.6G"
        }
    ]
}
ok: [192.xx.yy.19] => {
    "msg": [
        {
            "name": "sda",
            "size": "446.6G"
        },
        {
            "name": "sdb",
            "size": "446.6G"
        }
    ]
}
ok: [192.xx.yy.20] => {
    "msg": [
        {
            "name": "sda",
            "size": "446.6G"
        },
        {
            "name": "sdb",
            "size": "446.6G"
        }
    ]
}
```
