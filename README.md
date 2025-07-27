# Enterprise Server Provisioning with Ansible

## Overview

This project contains Ansible playbooks and configurations for provisioning enterprise servers with automated setup, file management, and reporting capabilities using Jinja2 templates and Ansible facts.

---

## Prerequisites

* Ansible installed on control node

* SSH key-based authentication configured

* Target servers accessible via SSH

---

## Server Inventory

### Inventory File (`inventory.ini`)

```ini
[web]
server1 ansible_host=54.169.176.160 ansible_user=ubuntu
server2 ansible_host=13.213.50.149 ansible_user=ubuntu

[web:vars]
ansible_ssh_private_key_file=~/.ssh/id_rsa
ansible_ssh_common_args='-o StrictHostKeyChecking=no'
```

---

## Jinja2 Report Template

### Simple Report Template (`templates/server_report.j2`)

```jinja2
{# Simple Server Management Report Template #}
=== SERVER MANAGEMENT REPORT ===
Generated: {{ ansible_date_time.date }} {{ ansible_date_time.time }} {{ ansible_date_time.tz }}

Server: {{ inventory_hostname }} ({{ ansible_host }})

OS Version: {{ ansible_distribution }} {{ ansible_distribution_version }}

Time Information:
- Control Node Time: {{ control_node_time.stdout }}
- Remote Host Time: {{ remote_host_time.stdout }}

File Management:
- Downloaded File: {{ download_filename }}
- File Exists on Remote: {{ 'Yes' if file_check.stat.exists else 'No' }}

================================
```

---

## Main Playbook

### Server Provisioning Playbook (`server_provisioning.yml`)

```yaml
---
- name: Web Server Provisioning
  hosts: web
  become: yes
  gather_facts: yes
  vars:
    download_url: "https://raw.githubusercontent.com/ansible/ansible/devel/README.md"
    download_filename: "README.md"
    target_directory: "task/devops-bootcamp/week3/ansible"
    local_download_path: "/tmp/{{ download_filename }}"
    
  tasks:
    - name: Gather system information
      setup:
      register: system_facts

    - name: Get current time on control node
      delegate_to: localhost
      command: date '+%Y-%m-%d %H:%M:%S %Z'
      register: control_node_time
      run_once: true
      vars:  
        ansible_become: false

    - name: Get current time on remote host
      command: date '+%Y-%m-%d %H:%M:%S %Z'
      register: remote_host_time

    - name: Display system information
      debug:
        msg: |
          Server: {{ inventory_hostname }}
          OS: {{ ansible_distribution }} {{ ansible_distribution_version }}
          Control Node Time: {{ control_node_time.stdout }}
          Remote Host Time: {{ remote_host_time.stdout }}

    - name: Check if Nginx is installed on Debian-based systems  
      ansible.builtin.shell: "dpkg -l | grep nginx"  
      register: nginx_check_debian  
      ignore_errors: yes  
      when: ansible_os_family == "Debian"  
  
    - name: Debug message for Nginx on Debian-based systems  
      ansible.builtin.debug:  
        msg: "Nginx is already installed on this Debian-based system."  
      when: nginx_check_debian.rc == 0 and ansible_os_family == "Debian"  
  
    - name: Install Nginx on Debian-based systems  
      ansible.builtin.apt:  
       update_cache: yes  
       name: nginx  
       state: present  
      when: nginx_check_debian.rc != 0 and ansible_os_family == "Debian"  
  
    - name: Ensure Nginx is enabled and started on Debian-based systems  
      ansible.builtin.service:  
        name: nginx  
        enabled: yes  
        state: started  
      when: ansible_os_family == "Debian"
    
    - name: Download file to control node
      delegate_to: localhost
      get_url:
          url: "{{ download_url }}"
          dest: "{{ local_download_path }}"
          mode: '0644'
      run_once: true
      vars:  
        ansible_become: false

    - name: Create target directory on control node
      delegate_to: localhost
      file:
        path: "{{ target_directory }}"
        state: directory
        mode: '0755'
      run_once: true
      vars:  
        ansible_become: false

    - name: Copy downloaded file to local target directory
      delegate_to: localhost
      copy:
        src: "{{ local_download_path }}"
        dest: "{{ target_directory }}/{{ download_filename }}"
        mode: '0644'
      run_once: true
      vars:  
        ansible_become: false

    - name: Create target directory on remote server
      file:
        path: "{{ target_directory }}"
        state: directory
        mode: '0755'

    - name: Copy file from control node to remote server
      copy:
        src: "{{ target_directory }}/{{ download_filename }}"
        dest: "{{ target_directory }}/{{ download_filename }}"
        mode: '0644'

    - name: Verify target directory exists on remote server
      stat:
        path: "{{ target_directory }}"
      register: target_dir_check

    - name: Verify file exists on remote server
      stat:
        path: "{{ target_directory }}/{{ download_filename }}"
        get_checksum: yes
      register: file_check

    - name: Generate timestamp for report
      delegate_to: localhost
      command: date '+%Y%m%d_%H%M%S'
      register: timestamp
      run_once: true
      vars:  
        ansible_become: false

    - name: Set report timestamp variable
      set_fact:
        report_timestamp: "{{ timestamp.stdout }}"

    - name: Create reports directory
      delegate_to: localhost
      file:
        path: reports
        state: directory
        mode: '0755'
      run_once: true
      vars:  
        ansible_become: false

    - name: Create templates directory
      delegate_to: localhost
      file:
        path: templates
        state: directory
        mode: '0755'
      run_once: true
      vars:  
        ansible_become: false

    - name: Generate comprehensive server report using Jinja2 template
      delegate_to: localhost
      template:
        src: templates/server_report.j2
        dest: "reports/{{ report_timestamp }}_server_mgmt_{{ inventory_hostname }}.txt"
        mode: '0644'
      vars:
        control_node_time: "{{ control_node_time }}"
        remote_host_time: "{{ remote_host_time }}"
        nginx_service_status: "{{ nginx_service_status }}"
        file_check: "{{ file_check }}"
        target_dir_check: "{{ target_dir_check }}"
        ansible_become: false

    - name: Display report generation status
      debug:
        msg: |
          ✓ Comprehensive server report generated successfully!
          📄 Report file: reports/{{ report_timestamp }}_server_mgmt_{{ inventory_hostname }}.txt
          📊 Report includes: System info, performance metrics, security status, and task results
          🕒 Generated at: {{ ansible_date_time.date }} {{ ansible_date_time.time }}
```

---

## Execution Instructions

### 1\. Setup SSH Key Authentication

```bash
# Generate a new SSH key (skip if you already have one)
ssh-keygen -t rsa -b 2048

# Copy your public key to both servers (for local VM)
ssh-copy-id -i ~/.ssh/id_rsa.pub ubuntu@192.168.0.108
ssh-copy-id -i ~/.ssh/id_rsa.pub ubuntu@192.168.0.109

# For ec2 instance
ssh-copy-id -f "-o IdentityFile <PATH TO PEM FILE>" ubuntu@<INSTANCE-PUBLIC-IP> #Debian
ssh-copy-id -f "-o IdentityFile <PATH TO PEM FILE>" ec2-user@<INSTANCE-PUBLIC-IP> #RedHat
```

### 2\. Create Template Directory and File

```bash
# Create templates directory
mkdir -p templates

# Copy the Jinja2 template content to templates/server_report.j2
# (Use the template content provided above)
```

### 3\. Test Connectivity

```bash
# Test SSH connectivity
ansible web -m ping

# Test with specific inventory
ansible -i inventory.ini web -m ping
```

### 4\. Run the Provisioning Playbook

```bash
# Execute the main playbook
ansible-playbook -i inventory.ini server_provisioning.yml

# Run with verbose output
ansible-playbook -i inventory.ini server_provisioning.yml -v

# Dry run (check mode)
ansible-playbook -i inventory.ini server_provisioning.yml --check
```

### 5\. Verify Results

```bash
# Check generated reports
ls -la reports/*_server_mgmt_*.txt

# View a sample report
cat reports/$(ls reports/ | head -1)

# Verify nginx installation on remote servers
ansible web -m service -a "name=nginx state=started"

# Check file existence on remote servers
ansible -i inventory.ini web -m stat -a "path=task/devops-bootcamp/week3/ansible/README.md"
```

---

## Enhanced Features

### Jinja2 Template Benefits

* **Dynamic Content**: Uses Ansible facts for real-time system information

* **Conditional Logic**: Shows warnings based on system metrics

* **Comprehensive Data**: Includes hardware, network, security, and performance metrics

* **Professional Format**: Well-structured, readable report format

* **Automated Analysis**: Provides recommendations based on system status

### Report Sections

* **Server Information**: Hardware, OS, network details

* **Time Synchronization**: Time drift analysis and recommendations

* **Service Management**: Nginx and system service status

* **File Operations**: Detailed file transfer verification

* **Security & Compliance**: User, SSH, and security status

* **Performance Metrics**: Load, memory, and swap usage

* **Task Summary**: Execution results and recommendations

---

## Troubleshooting

### Common Issues and Solutions

#### Missing Sudo Password Error

```bash
# Use become: yes in the playbook to avoid this issue
```

#### Template Not Found Error

```bash
# Ensure templates directory exists
mkdir -p templates

# Verify template file exists
ls -la templates/server_report.j2
```

#### Jinja2 Template Errors

```bash
# Test template syntax
ansible-playbook --syntax-check server_provisioning.yml

# Run with verbose output to see template processing
ansible-playbook -i inventory.ini server_provisioning.yml -vvv
```

---
