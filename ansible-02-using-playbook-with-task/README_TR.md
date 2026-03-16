# Hands-on Ansible-02: Playbook ile Görev Yönetimi

Bu hands-on eğitimin amacı, öğrencilere temel Ansible Playbook becerilerini kazandırmaktır.

## Öğrenim Hedefleri

Bu hands-on eğitimin sonunda öğrenciler:

- Ansible Playbook yapısını öğrenmiş olacak

## İçerik

- Part 1 - Ansible Kurulumu
- Part 2 - Ansible Playbooks

---

## Part 1 - Ansible Kurulumu

4 adet EC2 instance oluşturun ve aşağıdaki gibi isimlendirin. Security Group'ta SSH (22) ve HTTP (80) portlarını açık tutun.

1. **control node** — Amazon Linux 2023
2. **node1** — Amazon Linux 2023
3. **node2** — Amazon Linux 2023
4. **node3** — Ubuntu 22.04 LTS

Control node'a SSH ile bağlanın ve aşağıdaki komutları çalıştırın:

```bash
# Sistemdeki mevcut paketleri günceller.
sudo dnf update -y

# Ansible'ı kurar.
sudo dnf install ansible -y
```

### Kurulumu Doğrulama

```bash
# Kurulu Ansible sürümünü ve yapılandırma bilgilerini gösterir.
ansible --version
```

### Ansible Yapılandırması

```bash
# inventory.txt dosyasını oluşturur ve düzenlemek için açar.
vim inventory.txt
```

```ini
[webservers]
node1 ansible_host=<node1_ip> ansible_user=ec2-user
node2 ansible_host=<node2_ip> ansible_user=ec2-user

[ubuntuservers]
node3 ansible_host=<node3_ip> ansible_user=ubuntu

[all:vars]
ansible_ssh_private_key_file=/home/ec2-user/<pem_dosyasi>
```

```bash
# ansible.cfg dosyasını oluşturur ve düzenlemek için açar.
vim ansible.cfg
```

```ini
[defaults]
host_key_checking    = False
inventory            = inventory.txt
deprecation_warnings = False
interpreter_python   = auto_silent
```

> **Parametre Açıklamaları:**
> - `host_key_checking = False` : SSH bağlantısında "bu sunucuya daha önce bağlanmadınız" uyarısını atlar.
> - `inventory = inventory.txt` : Her komutta `-i inventory.txt` yazmak zorunda kalmayız.
> - `deprecation_warnings = False` : Eski kullanım uyarılarını gizler.
> - `interpreter_python = auto_silent` : Python yorumlayıcısını otomatik seçer, uyarı basmaz.

`.pem` dosyasını kendi bilgisayarınızdan control node'a kopyalayın:

```bash
# scp: SSH üzerinden güvenli dosya kopyalama.
# -i <pem>         : Bağlantı için kullanılacak kimlik anahtarı.
# İlk <pem>        : Kopyalanacak kaynak dosya.
# ec2-user@...     : Hedef sunucu kullanıcısı ve IP adresi.
# :/home/ec2-user  : Dosyanın kopyalanacağı hedef dizin.
scp -i <pem_dosyasi> <pem_dosyasi> ec2-user@<control_node_public_dns>:/home/ec2-user
```

---

## Part 2 - Ansible Playbooks

### Playbook Nedir?

Playbook, birden fazla görevi (task) sırayla tanımladığınız YAML formatındaki Ansible dosyasıdır. Ad-hoc komutların aksine karmaşık ve tekrar edilebilir otomasyon senaryoları için kullanılır.

```bash
# Tüm playbook'ların tutulacağı klasörü oluşturur ve içine girer.
mkdir playbooks && cd playbooks
```

---

### playbook1.yml — Bağlantı Testi

```bash
vim playbook1.yml
```

```yaml
---
- name: Test Connectivity        # Play'in açıklayıcı adı
  hosts: all                     # Tüm inventory host'larını hedef al
  tasks:
    - name: Ping test            # Task'ın açıklayıcı adı
      ansible.builtin.ping:      # SSH + Python bağlantısını test eden modül (ICMP değil)
```

```bash
# Playbook'u çalıştırır.
# Alternatif olarak ad-hoc ile de bağlantı test edilebilir:
ansible all -m ping -o

# Playbook'u çalıştır.
ansible-playbook playbook1.yml
```

---

### playbook2.yml — Dosya Kopyalama

`testfile1` adında bir metin dosyası oluşturun:

```bash
# testfile1 dosyasını oluşturur.
vim testfile1
# İçine "Hello Ondiaway" yazın, kaydedin.
```

```bash
vim playbook2.yml
```

```yaml
---
- name: Copy for linux
  hosts: webservers              # Yalnızca Amazon Linux node'larını hedef al
  tasks:
    - name: Copy your file to the webservers
      ansible.builtin.copy:
        src: /home/ec2-user/playbooks/testfile1   # Control node'daki kaynak dosya
        dest: /home/ec2-user/testfile1            # Managed node'daki hedef yol

- name: Copy for ubuntu
  hosts: ubuntuservers           # Yalnızca Ubuntu node'larını hedef al
  tasks:
    - name: Copy your file to the ubuntuservers
      ansible.builtin.copy:
        src: /home/ec2-user/playbooks/testfile1
        dest: /home/ubuntu/testfile1
        mode: u+rw,g-wx,o-rwx    # Dosya izinleri: sahip okuyabilir/yazabilir, grup ve diğerleri erişemez

- name: Copy for node1
  hosts: node1                   # Yalnızca node1'i hedef al
  tasks:
    - name: Copy using inline content
      ansible.builtin.copy:
        content: 'This is content of file2'   # Kaynak dosya yerine doğrudan içerik yazar
        dest: /home/ec2-user/testfile2

    - name: Create a new text file
      ansible.builtin.shell: "echo Hello World > /home/ec2-user/testfile3"
      # shell modülü: pipe, redirect (>) gibi shell operatörlerini destekler
```

```bash
# Playbook'u çalıştır.
ansible-playbook playbook2.yml

# Dosyaların kopyalandığını doğrulamak için node1'e bağlanın:
# ssh -i <pem> ec2-user@<node1_ip>
# ls -la /home/ec2-user/
```

---

### playbook3.yml — Apache Kurulumu

```bash
# Yum modülünün belgelerini incele (Amazon Linux için).
ansible-doc yum

# Apt modülünün belgelerini incele (Ubuntu için).
ansible-doc apt

vim playbook3.yml
```

```yaml
---
- name: Apache installation for webservers
  hosts: webservers
  tasks:
    - name: Install the latest version of Apache
      ansible.builtin.dnf:             # Amazon Linux 2023 paket yöneticisi
        name: httpd                    # Apache'nin Amazon Linux'taki paket adı
        state: latest                  # En güncel sürümü kur

    - name: Start Apache
      ansible.builtin.shell: "service httpd start"
      # httpd servisini başlatır; service modülü yerine shell kullanıldı

- name: Apache installation for ubuntuservers
  hosts: ubuntuservers
  tasks:
    - name: Update apt cache
      ansible.builtin.shell: "apt update -y"
      # Paket listesini günceller; kurulumdan önce gereklidir

    - name: Install the latest version of Apache
      ansible.builtin.apt:             # Ubuntu/Debian paket yöneticisi
        name: apache2                  # Apache'nin Ubuntu'daki paket adı
        state: latest
```

```bash
# -b (--become): root yetkisiyle çalıştırır (paket kurulumu için gerekli).
ansible-playbook -b playbook3.yml

# Aynı komutu tekrar çalıştırın → idempotency'i gözlemleyin.
# Zaten kurulu olan paketler için "changed: false" (yeşil) döner.
ansible-playbook -b playbook3.yml
```

> Tarayıcınızda `http://<node1_ip>` ve `http://<node2_ip>` adreslerini açarak Apache varsayılan sayfasını görün.

---

### playbook4.yml — Apache Kaldırma

```bash
vim playbook4.yml
```

```yaml
---
- name: Remove Apache from webservers
  hosts: webservers
  tasks:
    - name: Remove Apache
      ansible.builtin.dnf:
        name: httpd
        state: absent          # Paketi kaldır; kurulu değilse dokunma (idempotent)
        autoremove: yes        # httpd'ye bağımlı gereksiz paketleri de kaldırır

- name: Remove Apache from ubuntuservers
  hosts: ubuntuservers
  tasks:
    - name: Remove Apache
      ansible.builtin.apt:
        name: apache2
        state: absent
        autoremove: yes        # Bağımlı gereksiz paketleri de kaldırır
        purge: yes             # Konfigürasyon dosyalarını da siler (apt purge)
```

```bash
ansible-playbook -b playbook4.yml
```

---

### playbook5.yml — Apache + wget Kurulumu ve loop Kullanımı

```bash
vim playbook5.yml
```

```yaml
---
- name: Apache installation and configuration for ubuntuservers
  hosts: ubuntuservers
  tasks:
    - name: Install Apache
      ansible.builtin.apt:
        name: apache2
        state: latest

    - name: Deploy index.html
      ansible.builtin.copy:
        content: "<h1>Hello Ondiaway</h1>"   # HTML içeriğini doğrudan dosyaya yazar
        dest: /var/www/html/index.html        # Apache'nin varsayılan web dizini

    - name: Restart apache2
      ansible.builtin.service:
        name: apache2
        state: restarted   # Servisi yeniden başlatır (config değişikliği sonrası gerekli)
        enabled: yes       # Sistem açılışında otomatik başlamasını sağlar

- name: Apache and wget installation for webservers
  hosts: webservers
  tasks:
    - name: Install httpd and wget
      ansible.builtin.dnf:
        pkg: "{{ item }}"    # loop ile her seferinde farklı bir paket adı gelir
        state: present
      loop:                  # Listedeki her eleman için task'ı tekrar çalıştırır
        - httpd
        - wget

    - name: Enable and start httpd service
      ansible.builtin.service:
        name: httpd
        state: started
        enabled: true
```

```bash
ansible-playbook -b playbook5.yml
```

> `http://<node3_ip>` adresini tarayıcıda açın → "Hello Ondiaway" sayfasını görün.

---

### playbook6.yml — Apache + wget Kaldırma

```bash
vim playbook6.yml
```

```yaml
---
- name: Remove Apache from ubuntuservers
  hosts: ubuntuservers
  tasks:
    - name: Uninstall Apache
      ansible.builtin.apt:
        name: apache2
        state: absent
        update_cache: yes    # Kaldırmadan önce paket listesini günceller
        autoremove: yes
        purge: yes

- name: Remove Apache and wget from webservers
  hosts: webservers
  tasks:
    - name: Remove httpd and wget
      ansible.builtin.dnf:
        pkg: "{{ item }}"    # loop ile her paketi sırayla kaldırır
        state: absent
      loop:
        - httpd
        - wget
```

```bash
ansible-playbook -b playbook6.yml
```

---

### playbook7.yml — loop ve when ile Kullanıcı Oluşturma

```bash
vim playbook7.yml
```

```yaml
---
- name: Create users
  hosts: "*"               # Tüm host'ları hedef al (* = all ile aynı anlam)
  tasks:
    - name: Create users for RedHat OS family
      ansible.builtin.user:
        name: "{{ item }}"   # loop'tan gelen kullanıcı adı
        state: present
      loop:
        - recep
        - alex
        - james
        - oliver
      when: ansible_os_family == "RedHat"
      # when: koşul sağlanmıyorsa task SKIPPED olarak geçilir
      # ansible_os_family: Ansible'ın topladığı bir fact (sistem bilgisi)
      # RedHat ailesine girer: Amazon Linux, RHEL, CentOS, Fedora

    - name: Create users for SUSE OS family
      ansible.builtin.user:
        name: "{{ item }}"
        state: present
      loop:
        - david
        - polat
      when: ansible_os_family == "SUSE"

    - name: Create users for Debian OS family
      ansible.builtin.user:
        name: "{{ item }}"
        state: present
      loop:
        - tomy
        - robin
      when: ansible_os_family == "Debian" or ansible_distribution_version == "20.04"
      # 'or' ile birden fazla koşul birleştirilebilir
      # Debian ailesine girer: Ubuntu, Debian
```

```bash
ansible-playbook -b playbook7.yml
```

Kullanıcıların oluşturulduğunu doğrulayın:

```bash
# Tüm host'larda /etc/passwd dosyasını okur ve 'recep' kullanıcısını filtreler.
# /etc/passwd : Tüm kullanıcıların listelendiği dosya (şifresiz, herkese açık).
# /etc/shadow : Şifrelenmiş parola bilgilerini içerir (yalnızca root okuyabilir).
ansible all -b -m command -a "cat /etc/passwd" | grep 'recep'
```

---

## Özet: Önemli Kavramlar

### Sık Kullanılan Seçenekler

| Seçenek | Uzun Adı | Açıklama |
|---------|----------|----------|
| `-o` | `--one-line` | Çıktıyı tek satırda özetler |
| `-m` | `--module-name` | Kullanılacak modül adını belirtir |
| `-b` | `--become` | Elevated (root) yetki kullanır |
| `-a` | `--args` | Modüle gönderilecek argümanlar |
| `--list-hosts` | — | Eşleşen host listesini gösterir, komut çalıştırmaz |
| `-i` | `--inventory` | Komut satırında inventory dosyasını override eder |

### Idempotency — Modül Karşılaştırması

| Modül | Idempotent | Açıklama |
|-------|-----------|----------|
| `shell` | ❌ Hayır | Her çalıştırmada komutu tekrar işletir |
| `yum` | ✅ Evet | Paket zaten varsa hiçbir şey yapmaz |
| `dnf` | ✅ Evet | Paket zaten varsa hiçbir şey yapmaz |
| `apt` | ✅ Evet | Paket zaten varsa hiçbir şey yapmaz |
| `package` | ✅ Evet | OS bağımsız; paket zaten varsa hiçbir şey yapmaz |

### Kullanıcı Dosyaları

| Dosya | Erişim | İçerik |
|-------|--------|--------|
| `/etc/passwd` | Herkese açık | Kullanıcı adı, UID, GID, home dizini |
| `/etc/shadow` | Yalnızca root | Şifrelenmiş parola bilgileri |
