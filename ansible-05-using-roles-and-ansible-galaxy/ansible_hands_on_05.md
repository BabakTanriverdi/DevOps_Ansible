# Hands-on Ansible-05: Roles Kullanımı

Bu hands-on eğitimin amacı, öğrencilere Ansible role kavramını ve kullanımını öğretmektir.

## Öğrenim Hedefleri

Bu hands-on eğitimin sonunda öğrenciler:

- Ansible role'ün ne olduğunu açıklayabilecek
- Role nasıl oluşturulur, bulunur ve kullanılır öğrenmiş olacak

## İçerik

- Part 1 - Ansible Kurulumu
- Part 2 - Ansible Roles Kullanımı
- Part 3 - Ansible Galaxy'den Role Kullanımı
- Opsiyonel - Custom AMI için Role Yönetimi

---

## Part 1 - Ansible Kurulumu

> **Bu bölümde ne yapıyoruz?**
> Önceki derslerde olduğu gibi altyapıyı kuruyoruz. Bu sefer farklı olarak iki farklı işletim sistemli web sunucusu (Red Hat + Ubuntu) kullanıyoruz — bu, role'lerin çoklu OS desteğini görmemizi sağlayacak.

3 adet EC2 instance oluşturun:

| Instance | İşletim Sistemi | Açık Portlar |
|----------|----------------|--------------|
| control node | Amazon Linux 2023 | 22 (SSH) |
| web_server_1 | Amazon Linux 2023 (Red Hat) | 22 (SSH), 80 (HTTP) |
| web_server_2 | Ubuntu 22.04 | 22 (SSH), 80 (HTTP) |

```bash
# Sistemdeki mevcut paketleri günceller.
sudo dnf update -y

# Ansible'ı kurar.
sudo dnf install ansible -y

# Kurulumu doğrular.
ansible --version
```

`.pem` dosyasını control node'a kopyalayın:

```bash
# scp: SSH üzerinden güvenli dosya kopyalama.
# Kendi bilgisayarınızda çalıştırın; IP adresini kendi control node IP'nizle değiştirin.
scp -i ~/.ssh/<pem_dosyasi>.pem ~/.ssh/<pem_dosyasi>.pem ec2-user@<control_node_ip>:/home/ec2-user
```

```bash
# Çalışma dizinini oluşturur ve içine girer.
mkdir working-with-roles
cd working-with-roles
```

`inventory.txt` dosyasını oluşturun:

```bash
vi inventory.txt
```

```ini
[servers]
web_server_1  ansible_host=<web_server_1_ip>  ansible_user=ec2-user  ansible_ssh_private_key_file=~/<pem_dosyasi>
web_server_2  ansible_host=<web_server_2_ip>  ansible_user=ubuntu    ansible_ssh_private_key_file=~/<pem_dosyasi>
```

`ansible.cfg` dosyasını oluşturun:

```bash
vi ansible.cfg
```

```ini
[defaults]
host_key_checking = False
inventory         = inventory.txt
interpreter_python = auto_silent
roles_path        = /home/ec2-user/ansible/roles/
# roles_path: Ansible'ın role'leri arayacağı dizin.
# Burada oluşturduğunuz veya Galaxy'den indirdiğiniz tüm role'ler bu dizinde saklanır.
```

Bağlantıyı doğrulayın:

```bash
touch ping-playbook.yml
```

```yaml
---
- name: Ping them all
  hosts: all
  tasks:
    - name: Pinging
      ansible.builtin.ping:
```

```bash
ansible-playbook ping-playbook.yml
```

---

## Part 2 - Ansible Roles Kullanımı

> **Bu bölümde ne yapıyoruz?**
> Role nedir ve neden kullanırız?
>
> Şimdiye kadar tüm task'ları tek bir playbook dosyasına yazdık. Projeniz büyüdükçe bu dosya yüzlerce satıra ulaşır ve yönetilmesi zorlaşır. **Role**, bir otomasyon görevini (apache kurulumu, mysql kurulumu vb.) tasks, handlers, variables, templates gibi standart bir klasör yapısıyla paketlemenizi sağlar. Böylece:
> - Aynı role'ü farklı projelerde yeniden kullanabilirsiniz.
> - Takım arkadaşlarınız role'ü kolayca anlayabilir.
> - Ansible Galaxy üzerinden topluluk role'lerini projenize dahil edebilirsiniz.

### Role Oluşturma

```bash
# ansible-galaxy init: Standart role dizin yapısını otomatik oluşturur.
# apache       : Oluşturulacak role'ün adı.
# --init-path  : Role'ün oluşturulacağı dizin. Olmayan dizini kendisi olusturur
ansible-galaxy init apache --init-path ../ansible/roles/
```

Oluşturulan yapıyı inceleyin:

```bash
cd /home/ec2-user/ansible/roles/apache

# tree: Dizin yapısını ağaç görünümünde gösterir.
sudo dnf install tree -y
tree
```

Beklenen çıktı:
```
apache/
├── defaults/
│   └── main.yml      ← Varsayılan değişkenler (en düşük öncelikli)
├── handlers/
│   └── main.yml      ← Handler tanımları (notify ile tetiklenir)
├── meta/
│   └── main.yml      ← Role metadata: yazar, lisans, bağımlılıklar
├── tasks/
│   └── main.yml      ← Ana task listesi (buraya yazıyoruz)
├── templates/         ← Jinja2 şablon dosyaları (.j2)
├── tests/
│   ├── inventory
│   └── test.yml
└── vars/
    └── main.yml      ← Değişkenler (defaults'tan yüksek öncelikli)
```

### tasks/main.yml Dosyasını Düzenleme

```bash
vi tasks/main.yml
```

```yaml
- name: Install Apache
  ansible.builtin.yum:
    name: httpd      # Apache'nin Red Hat/Amazon Linux'taki paket adı
    state: latest    # En güncel sürümü kur

- name: Deploy index.html
  ansible.builtin.copy:
    content: "<h1>Hello Ansible</h1>"   # HTML içeriğini doğrudan dosyaya yazar
    dest: /var/www/html/index.html      # Apache'nin varsayılan web dizini

- name: Restart Apache
  ansible.builtin.service:
    name: httpd
    state: restarted   # Servisi yeniden başlatır
    enabled: yes       # Sistem açılışında otomatik başlamasını sağlar
```

### Role'ü Playbook'ta Kullanma

```bash
cd /home/ec2-user/working-with-roles/
vi role1.yml
```

```yaml
---
- name: Install and Start Apache
  hosts: web_server_1
  become: yes      # Root yetkisiyle çalıştır (paket kurulumu için gerekli)
  roles:
    - apache
    # roles: Task listesi yerine role adı yazılır.
    # Ansible, ansible.cfg'deki roles_path dizininde 'apache' klasörünü arar
    # ve tasks/main.yml dosyasını otomatik çalıştırır.
```

```bash
ansible-playbook role1.yml
```

> Tarayıcınızda `http://<web_server_1_ip>` adresini açın → "Hello Ansible" sayfasını görün.

---

## Part 3 - Ansible Galaxy'den Role Kullanımı

> **Bu bölümde ne yapıyoruz?**
> Her şeyi sıfırdan yazmak zorunda değilsiniz. Ansible Galaxy, topluluk tarafından yazılmış ve test edilmiş binlerce hazır role barındırır. Bu bölümde Galaxy'den nginx role'ünü bulup kullanmayı öğreniyoruz.
>
> **Collection ile Role farkı:**
> - **Role**: Belirli bir görevi (nginx kurulumu gibi) yapan yeniden kullanılabilir yapı.
> - **Collection**: Birden fazla role, modül ve plugin'i bir arada paketleyen daha büyük yapı.

### Galaxy'de Role Arama

```bash
# Galaxy'de nginx ile ilgili tüm role'leri listeler.
# 1000'den fazla sonuç döner; filtreleyelim.
ansible-galaxy search nginx
```

```bash
# --platform EL: Enterprise Linux (CentOS/RHEL/Amazon Linux) platformunu filtreler.
# EL = Enterprise Linux
ansible-galaxy search nginx --platform EL
```

```bash
# Sonuçları geerlingguy ile filtreleyelim.
# geerlingguy: En çok indirilen, güvenilir Ansible role yazarlarından biridir.
ansible-galaxy search nginx --platform EL | grep geerl
```

```
geerlingguy.nginx    Nginx installation for Linux, FreeBSD and OpenBSD.
geerlingguy.php      PHP for RedHat/CentOS/Fedora/Debian/Ubuntu.
```

### Role'ü İndirme ve İnceleme

```bash
# geerlingguy.nginx role'ünü Galaxy'den indirir.
# Role, ansible.cfg'deki roles_path dizinine kurulur.
ansible-galaxy install geerlingguy.nginx
```

```
- downloading role 'nginx', owned by geerlingguy
- extracting geerlingguy.nginx to /home/ec2-user/.ansible/roles/geerlingguy.nginx
- geerlingguy.nginx (2.8.0) was installed successfully
```

Role yapısını inceleyin:

```bash
cd /home/ec2-user/.ansible/roles/geerlingguy.nginx
ls
# defaults  handlers  LICENSE  meta  molecule  README.md  tasks  templates  vars

cd tasks && ls
# main.yml  setup-Debian.yml  setup-FreeBSD.yml  setup-OpenBSD.yml
# setup-Archlinux.yml  setup-RedHat.yml  setup-Ubuntu.yml  vhosts.yml
```

`tasks/main.yml` dosyasını inceleyin:

```bash
vi main.yml
```

```yaml
---
- name: Include OS-specific variables.
  include_vars: "{{ ansible_os_family }}.yml"
  # ansible_os_family: Ansible fact'i — işletim sistemi ailesini döndürür.
  # RedHat, Debian, Ubuntu vb. için farklı değişken dosyaları yükler.

- name: Define nginx_user.
  set_fact:
    nginx_user: "{{ __nginx_user }}"
  when: nginx_user is not defined
  # set_fact: Çalışma zamanında yeni bir değişken tanımlar.
  # when: nginx_user zaten tanımlıysa bu task'ı atla.

- include_tasks: setup-RedHat.yml
  when: ansible_os_family == 'RedHat'
  # Her OS için ayrı kurulum dosyası var; doğru olanı otomatik seçilir.

- include_tasks: setup-Ubuntu.yml
  when: ansible_distribution == 'Ubuntu'

- include_tasks: setup-Debian.yml
  when: ansible_os_family == 'Debian'

- include_tasks: setup-FreeBSD.yml
  when: ansible_os_family == 'FreeBSD'

- include_tasks: setup-OpenBSD.yml
  when: ansible_os_family == 'OpenBSD'

- include_tasks: setup-Archlinux.yml
  when: ansible_os_family == 'Archlinux'

- import_tasks: vhosts.yml
  # import_tasks vs include_tasks:
  # import_tasks : Derleme zamanında dahil edilir (statik).
  # include_tasks: Çalışma zamanında dahil edilir (dinamik, when ile kullanılabilir).

- name: Copy nginx configuration in place.
  template:
    src: "{{ nginx_conf_template }}"
    dest: "{{ nginx_conf_file_path }}"
    owner: root
    group: "{{ root_group }}"
    mode: 0644
  notify:
    - reload nginx
  # template modülü: Jinja2 şablon dosyasını işleyerek hedefe kopyalar.
  # notify: Konfigürasyon değişirse nginx'i yeniden yükler.
```

### Galaxy Role'ünü Playbook'ta Kullanma

```bash
cd /home/ec2-user/working-with-roles/
vi playbook-nginx.yml
```

```yaml
---
- name: Use Galaxy nginx role
  hosts: web_server_2    # Ubuntu sunucusunu hedef al
  become: true

  roles:
    - geerlingguy.nginx  # Galaxy'den indirilen role adı: <kullanici>.<role>
```

```bash
ansible-playbook playbook-nginx.yml
```

Kurulu role'leri listeleyin:

```bash
# Sistemde kurulu tüm role'leri ve sürümlerini listeler.
ansible-galaxy list
```

```
- apache, (unknown version)
- geerlingguy.nginx, 2.8.0
```

---

## Opsiyonel - Custom AMI için Role Yönetimi

> **Bu bölümde ne yapıyoruz?**
> Gerçek bir senaryo: Docker ve Prometheus yüklü bir özel AMI hazırlamak istiyorsunuz. Bu AMI'yi her 6 ayda bir güncelleyeceksiniz. Role'ler sayesinde yazılım versiyonlarını tek bir dosyadan yönetebilirsiniz.

Yeni bir Ubuntu 24.04 EC2 instance oluşturun (AMI: `ami-04a81a99f5ec58529`).

```bash
# git: Role'leri GitHub'dan çekmek için gerekli.
sudo dnf install git -y
```

`role_requirements.yml` dosyasını oluşturun:

```yaml
# Bu dosya, projenin ihtiyaç duyduğu tüm role'leri ve sürümlerini tanımlar.
# 'ansible-galaxy install -r' komutu bu dosyayı okuyarak hepsini tek seferde kurar.

- src: git+https://github.com/geerlingguy/ansible-role-docker
  name: docker
  version: 7.4.3           # Belirli bir sürüm pinler; güncelleme kontrolünüzde olur

- src: git+https://github.com/geerlingguy/ansible-role-ntp
  version: 2.5.0
  name: ansible-role-ntp   # NTP: Network Time Protocol — sunucu saatlerini senkronize eder

- src: git+https://github.com/UnderGreen/ansible-prometheus-node-exporter
  version: v1.5.1          # Prometheus: sistem metriklerini toplar ve izleme için sunar
```

```bash
# -r (--roles-file): role_requirements.yml dosyasındaki tüm role'leri kurar.
# Tek komutla tüm bağımlılıkları yükler.
ansible-galaxy install -r role_requirements.yml
```

```bash
# Kurulumu doğrulayın; tüm role'ler listede görünmeli.
ansible-galaxy list
```

Ortak görevler için `common` role oluşturun:

```bash
# common: Tüm instance'lara uygulanacak temel görevleri içerir.
ansible-galaxy init /home/ec2-user/ansible/roles/common
```

`common/tasks/main.yml` dosyasını düzenleyin:

```yaml
---
- name: Common Tasks
  ansible.builtin.debug:
    msg: Common Task Triggered    # Hangi role'ün çalıştığını ekrana basar

- name: Fix dpkg
  ansible.builtin.command: dpkg --configure -a
  # dpkg --configure -a: Yarım kalmış paket kurulumlarını tamamlar.
  # NTP kurulumundan önce çalıştırılmalı; aksi halde hata verebilir.

- name: Update apt
  ansible.builtin.apt:
    upgrade: dist      # Tüm paketleri günceller; bağımlılık çakışmalarını çözer
    update_cache: yes  # Paket listesini günceller

- name: Install NTP package
  ansible.builtin.apt:
    name: ntp
    state: present
```

Ana playbook `instance_image.yml`:

```yaml
---
- hosts: instance_image   # inventory'de bu alias ile tanımladığınız Ubuntu instance
  become: yes

  roles:
    - common                                              # Önce ortak görevler
    - { role: ansible-role-ntp, ntp_timezone: UTC }      # NTP; timezone değişkeni ile özelleştiriliyor
    - docker                                             # Docker kurulumu
    - ansible-prometheus-node-exporter                   # Prometheus node exporter

  tasks:
    - ansible.builtin.import_tasks: './slack.yml'
    # import_tasks: Slack bildirim task'larını ayrı dosyadan dahil eder.
    # import (statik) kullandık çünkü bu task her zaman çalışmalı.
```

> **Not:** `instance_image` alias'ını inventory.txt dosyanıza ekleyin:
> ```ini
> instance_image ansible_host=<ubuntu_instance_ip> ansible_user=ubuntu ansible_ssh_private_key_file=~/<pem_dosyasi>
> ```

`slack.yml` — Deployment tamamlandığında Slack bildirimi:

```yaml
---
- name: Send Slack notification
  community.general.slack:
    token: "{{ slack_token }}"
    msg: '{{ inventory_hostname }} Deployed with Ansible'
    channel: "{{ slack_channel }}"
    username: "{{ slack_username }}"
  delegate_to: localhost
  # delegate_to: localhost: Bu task managed node'da değil, control node'da çalışır.
  # Slack API'ye control node'dan istek atılır.

  run_once: true
  # run_once: Kaç host olursa olsun bu task yalnızca bir kez çalışır.
  # 10 sunucu deploy etseniz bile tek bir Slack mesajı gönderilir.

  become: no
  # become: no: Slack bildirimi için root yetkisi gerekmez.

  when: inventory_hostname == ansible_play_hosts_all[-1]
  # when: Yalnızca son host için çalışır.
  # ansible_play_hosts_all[-1]: Play'deki son host'un adı.
  # Tüm deployment bittikten sonra bildirim gönderir.

  vars:
    slack_token: "YOUR/TOKEN"        # Slack app token'ınız
    slack_channel: "#class-chat-tr"  # Bildirim gidecek kanal
    slack_username: "Ansible"        # Botun görünen adı
```

```bash
ansible-playbook instance_image.yml
```

---

## Özet: Role Yapısı ve Kullanım Senaryoları

```
roles/
└── apache/
    ├── tasks/main.yml      ← Ne yapılacak (zorunlu)
    ├── handlers/main.yml   ← notify ile tetiklenen aksiyonlar
    ├── defaults/main.yml   ← Varsayılan değişkenler (en düşük öncelik)
    ├── vars/main.yml       ← Sabit değişkenler (defaults'tan yüksek öncelik)
    ├── templates/          ← Jinja2 şablon dosyaları (.j2)
    ├── files/              ← Statik dosyalar (copy modülü için)
    └── meta/main.yml       ← Role metadata ve bağımlılıklar
```

| Yöntem | Komut | Ne Zaman Kullanılır? |
|--------|-------|----------------------|
| Kendi role'ünü oluştur | `ansible-galaxy init <role>` | Kuruma özgü, tekrar kullanılacak görevler |
| Galaxy'den tek role kur | `ansible-galaxy install <role>` | Hazır, topluluk destekli role'ler |
| Gereksinim dosyasından kur | `ansible-galaxy install -r requirements.yml` | Çok role'lü projeler; versiyon yönetimi |
| Kurulu role'leri listele | `ansible-galaxy list` | Mevcut role'leri görme |

## Referans Linkler

- [Ansible Galaxy](https://galaxy.ansible.com/)
- [Ansible Roles Dokümantasyonu](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html)
- [geerlingguy.nginx](https://galaxy.ansible.com/geerlingguy/nginx)
- [Network Time Protocol (NTP)](https://en.wikipedia.org/wiki/Network_Time_Protocol)
