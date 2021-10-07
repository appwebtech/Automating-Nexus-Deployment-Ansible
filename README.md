# Automating Nexus Deployment with Ansible

## Introduction

Instead of creating a cloud server and SSH'ing into it to download Nexus binaries, unpack and eventually run Nexus, I will be automating this processes with Ansible.

## Installation

I will write the first two playbooks. One for installing java and net tools and the other for downloading Nexus binaries, then run it.

```yaml
---
- name: Install java and net-tools
  hosts: 178.62.13.23
  tasks:
    - name: Update repo and cache
      apt: update_cache=yes force_apt_get=yes cache_valid_time=3600
    - name: Install Java 8
      apt: name=openjdk-8-jre-headless
    - name: Install net-tools
      apt: name=net-tools

- name: Download and unpack Nexus installer
  hosts: 178.62.13.23
  tasks:
    - name: Download Nexus
      get_url:
        url: https://download.sonatype.com/nexus/3/latest-unix.tar.gz
        dest: /opt/
```

![playbook-1](./images/image-1.png)

</hr>

![verify-installation](./images/image-2.png)

Next step is to untar the downloaded nexus archive.

```yaml
    - name: Untar Nexus installer
      unarchive:
        src: /opt/nexus-3.34.1-01-unix.tar.gz
        dest: /opt/
        remote_src: yes
```

Because there are multiple versions of binaries to download and you never know which one you have till you SSH in the machine, you could actually configure the playbook so that the terminal will throw a "debug" message of a JSON with metadata of what you are getting.

```yaml
---
- name: Install java and net-tools
  hosts: 178.62.13.23
  tasks:
    - name: Update repo and cache
      apt: update_cache=yes force_apt_get=yes cache_valid_time=3600
    - name: Install Java 8
      apt: name=openjdk-8-jre-headless
    - name: Install net-tools
      apt: name=net-tools

- name: Download and unpack Nexus installer
  hosts: 178.62.13.23
  tasks:
    - name: Download Nexus
      get_url:
        url: https://download.sonatype.com/nexus/3/latest-unix.tar.gz
        dest: /opt/
      register: download_result
    - debug: msg={{download_result}}
#    - name: Untar Nexus installer
#      unarchive:
#        src: /opt/nexus-3.34.1-01-unix.tar.gz
#        dest: /opt/
#        remote_src: yes
```
![metadata](./images/image-3.png)

It's always good to know what you are installing, especially versions as they usually come in handy. Next I'll run the installation again without the debugger and see what is happening. Below is the second play.

```yaml
- name: Download and unpack Nexus installer
  hosts: 178.62.13.23
  tasks:
    - name: Download Nexus
      get_url:
        url: https://download.sonatype.com/nexus/3/latest-unix.tar.gz
        dest: /opt/
      register: download_result
    - name: Untar Nexus installer
      unarchive:
        src: "{{download_result.dest}}"
        dest: /opt/
        remote_src: yes
```

Both download and unpacking was successful.

![unpacked-auto](./images/image-4.png)

If I want to debug and view the contents of the nexus folder (/opt/nexus-3.34.1-01) as an output then I can add the following;

```yaml
    - name: Find Nexus Folder
      find:
        paths: /opt/
        pattern: "nexus-*"
        file_type: directory
      register: find_result
    - debug: msg={{find_result}}
 #   - name: Rename nexus folder
 #     shell: mv {{find_result.files[0].path}} /opt/nexus
```

![nexus-content](./images/image-5.png)

Un-commenting the last two lines in the previous code and re-running the playbook will rename the folder to nexus

![rename-folder](./images/image-6.png)

It's possible to configure the playbook in a way that instead of throwing an error whilst copying to an already existing folder with files, it would actually skip it. I'm adding the modules I've changed instead of pasting the whole playbook.

```yaml
    - name: Check if folder already exists
      stat:
        path: /opt/nexus
      register: stat_result
    - name: Rename nexus folder
      shell: mv {{find_result.files[0].path}} /opt/nexus
      when: not stat_result.stat.exists
```

![skipping](./images/image-8.png)