# 📘 Ansible Idempotency Rehberi

> **Idempotency**, bir işlemin bir kez ya da birden fazla kez çalıştırılmasının aynı sonucu vermesi anlamına gelir.  
> Ansible'ın gücü buradan gelir: altyapıyı **istenen duruma** taşır, zaten o durumdaysa **dokunmaz**.



---

## 📑 İçindekiler

1. [Idempotency Nedir?](#idempotency-nedir)
2. [Idempotent Modüller](#idempotent-modüller)
3. [Non-Idempotent Modüller](#non-idempotent-modüller)
4. [`creates` / `removes` Parametresi](#creates--removes-parametresi)
5. [`register` + `changed_when` Kullanımı](#register--changed_when-kullanımı)
6. [`failed_when` Kullanımı](#failed_when-kullanımı)
7. [Gerçek Dünya Örnekleri](#gerçek-dünya-örnekleri)
8. [İyi Pratikler ve Altın Kurallar](#iyi-pratikler-ve-altın-kurallar)

---

## Idempotency Nedir?

```
İlk çalıştırma:
  Sunucu mevcut değil → Oluştur → changed: true

İkinci çalıştırma:
  Sunucu zaten var → Dokunma → changed: false  ✅

Üçüncü çalıştırma:
  Sunucu zaten var → Dokunma → changed: false  ✅
```

Ansible playbook'u **kaç kez çalıştırırsanız çalıştırın**, sonuç her zaman aynı olmalıdır.  
Bu, özellikle CI/CD pipeline'larında ve büyük altyapı yönetiminde kritik öneme sahiptir.

---

## ✅ Idempotent Modüller

Her çalıştırıldığında önce mevcut durumu kontrol eder, fark varsa değişiklik yapar, yoksa geçer.

### Dosya ve Dizin Yönetimi

| Modül | Açıklama | Örnek |
|---|---|---|
| `file` | Dosya/dizin/symlink oluşturur veya siler | `state: present/absent/directory/link` |
| `copy` | Dosya kopyalar; içerik aynıysa dokunmaz | Yerel → Remote |
| `template` | Jinja2 şablonu render eder | `.j2` dosyaları |
| `lineinfile` | Dosyaya satır ekler; varsa eklemez | Config satırı yönetimi |
| `blockinfile` | Dosyaya blok ekler; varsa eklemez | Çok satırlı config blokları |
| `replace` | Regex ile içerik değiştirir | Toplu find-replace |
| `stat` | Dosya/dizin bilgisi toplar | Kontrol amaçlı |
| `fetch` | Remote'dan yerel'e dosya çeker | Log toplama |

```yaml
# file modülü örneği
- name: Log dizini oluştur
  file:
    path: /var/log/myapp
    state: directory
    owner: myapp
    group: myapp
    mode: '0755'

# template modülü örneği
- name: Nginx config dosyasını yaz
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    owner: root
    mode: '0644'
  notify: restart nginx

# lineinfile örneği
- name: SSH root login'i kapat
  lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^PermitRootLogin'
    line: 'PermitRootLogin no'
  notify: restart sshd
```

---

### Paket Yönetimi

| Modül | Platform | Açıklama |
|---|---|---|
| `package` | Tüm platformlar | Genel paket yöneticisi |
| `apt` | Debian/Ubuntu | APT paket yöneticisi |
| `yum` | RHEL/CentOS 7 | YUM paket yöneticisi |
| `dnf` | RHEL/CentOS 8+ | DNF paket yöneticisi |
| `pip` | Python | Python paket yöneticisi |
| `snap` | Ubuntu | Snap paket yöneticisi |

```yaml
# apt ile paket kurulumu
- name: Web sunucu paketlerini kur
  apt:
    name:
      - nginx
      - curl
      - vim
    state: present
    update_cache: yes

# pip ile Python paketi
- name: Python bağımlılıklarını kur
  pip:
    name: "{{ item }}"
    state: present
  loop:
    - flask
    - gunicorn
    - psycopg2

# Paket kaldırma
- name: Eski Apache'yi kaldır
  package:
    name: apache2
    state: absent
```

---

### Servis Yönetimi

| Modül | Açıklama |
|---|---|
| `service` | Genel servis yönetimi |
| `systemd` | Systemd servis yönetimi (daha gelişmiş) |

```yaml
# systemd ile servis yönetimi
- name: Nginx servisini başlat ve enable et
  systemd:
    name: nginx
    state: started
    enabled: yes
    daemon_reload: yes

# Handler olarak kullanım
handlers:
  - name: restart nginx
    systemd:
      name: nginx
      state: restarted
```

---

### Kullanıcı ve Grup Yönetimi

```yaml
# Kullanıcı oluşturma
- name: Deploy kullanıcısı oluştur
  user:
    name: deploy
    uid: 1500
    group: deploy
    groups:
      - sudo
      - docker
    shell: /bin/bash
    home: /home/deploy
    create_home: yes
    state: present

# SSH authorized key
- name: Deploy kullanıcısına SSH key ekle
  authorized_key:
    user: deploy
    key: "{{ lookup('file', '~/.ssh/id_rsa.pub') }}"
    state: present
```

---

### Ağ ve Güvenlik Duvarı

```yaml
# UFW güvenlik duvarı
- name: HTTP ve HTTPS'e izin ver
  ufw:
    rule: allow
    port: "{{ item }}"
    proto: tcp
  loop:
    - '80'
    - '443'

# Firewalld
- name: HTTP servisine izin ver
  firewalld:
    service: http
    permanent: yes
    state: enabled
    immediate: yes
```

---

### Diğer Önemli Idempotent Modüller

```yaml
# cron - Cron job yönetimi
- name: Yedekleme cron job'u ekle
  cron:
    name: "daily backup"
    minute: "0"
    hour: "2"
    job: "/usr/local/bin/backup.sh"
    user: root

# git - Repository yönetimi
- name: Uygulama kodunu çek
  git:
    repo: https://github.com/myorg/myapp.git
    dest: /opt/myapp
    version: main
    force: no

# mount - Disk mount işlemleri
- name: NFS share'i bağla
  mount:
    path: /mnt/data
    src: 192.168.1.10:/data
    fstype: nfs
    opts: defaults
    state: mounted

# sysctl - Kernel parametreleri
- name: IP forwarding'i aktifleştir
  sysctl:
    name: net.ipv4.ip_forward
    value: '1'
    sysctl_set: yes
    reload: yes
```

---

## ⚠️ Non-Idempotent Modüller

Her çalıştırıldığında işlemi gerçekleştirir, önceki durumu kontrol etmez.

| Modül | Neden Non-Idempotent? | Risk |
|---|---|---|
| `command` | Her seferinde çalıştırır | Duplicate işlem |
| `shell` | Her seferinde çalıştırır | Duplicate işlem |
| `raw` | Ham SSH komutu, kontrol yok | Tahmin edilemez sonuç |
| `script` | Scripti her seferinde çalıştırır | Duplicate kurulum |
| `uri` (POST/DELETE) | HTTP isteği atar | Duplicate kayıt |
| `expect` | Interaktif komut çalıştırır | Her seferinde tekrarlanır |

```yaml
# ❌ KÖTÜ - Her çalıştırmada kullanıcı oluşturmaya çalışır
- name: Kullanıcı oluştur
  command: useradd myuser

# ❌ KÖTÜ - Her çalıştırmada aynı satırı tekrar ekler
- name: PATH ekle
  shell: echo 'export PATH=$PATH:/opt/bin' >> ~/.bashrc

# ✅ İYİ - Idempotent modül kullan
- name: Kullanıcı oluştur
  user:
    name: myuser
    state: present

# ✅ İYİ - lineinfile kullan
- name: PATH ekle
  lineinfile:
    path: ~/.bashrc
    line: 'export PATH=$PATH:/opt/bin'
```

---

## `creates` / `removes` Parametresi

Non-idempotent modülleri `creates` veya `removes` parametresiyle idempotent hale getirme yöntemi.

### `creates` Parametresi

```
Ansible çalışır
       │
       ▼
creates'te belirtilen dosya var mı?
       │
   ┌───┴───┐
  EVET   HAYIR
   │       │
   ▼       ▼
 SKIP   Komutu çalıştır
(changed: false)   │
                   ▼
     (Dosya oluşturulmalı)
     changed: true
```

```yaml
# Temel kullanım
- name: Setup scriptini sadece bir kez çalıştır
  command: /usr/bin/setup.sh
  args:
    creates: /etc/setup.done

# Script çalıştıktan sonra flag dosyası oluştur
- name: Kurulum scriptini çalıştır
  command: /opt/install.sh
  args:
    creates: /opt/.install_complete

- name: Kurulum tamamlandı flag'i oluştur
  file:
    path: /opt/.install_complete
    state: touch
  when: not ansible_check_mode
```

### `removes` Parametresi

`creates`'in tersidir. Belirtilen dosya **varsa** komutu çalıştırır.

```yaml
# Dosya varsa temizlik yap
- name: Eski temp dosyaları temizle
  command: /usr/bin/cleanup.sh
  args:
    removes: /tmp/old_data

# Geçici kurulum dosyasını sil
- name: Installer dosyasını çalıştır ve sil
  shell: /tmp/installer.sh && rm /tmp/installer.sh
  args:
    removes: /tmp/installer.sh
```

### Gerçek Dünya: Uygulama Kurulumu

```yaml
---
- name: MyApp Kurulum Playbook
  hosts: app_servers
  become: yes

  tasks:
    - name: Installer'ı indir
      get_url:
        url: https://releases.myapp.io/v2.5/install.sh
        dest: /tmp/myapp_install.sh
        mode: '0755'

    - name: MyApp'ı kur (sadece kurulu değilse)
      command: /tmp/myapp_install.sh --silent
      args:
        creates: /opt/myapp/bin/myapp

    - name: Installer'ı temizle
      file:
        path: /tmp/myapp_install.sh
        state: absent

    - name: Servis dosyasını oluştur
      template:
        src: myapp.service.j2
        dest: /etc/systemd/system/myapp.service

    - name: Servisi başlat
      systemd:
        name: myapp
        state: started
        enabled: yes
        daemon_reload: yes
```

---

## `register` + `changed_when` Kullanımı

Komutun çıktısını değişkene kaydedip, `changed` durumunu programatik olarak kontrol etme yöntemi.

### `register` Değişkeninin Yapısı

```yaml
result:
  rc: 0              # Return code (0 = başarılı)
  stdout: "..."      # Standart çıktı
  stderr: "..."      # Hata çıktısı
  stdout_lines: []   # stdout satır satır liste
  stderr_lines: []   # stderr satır satır liste
  changed: true      # Değişiklik durumu
  failed: false      # Hata durumu
  cmd: "..."         # Çalıştırılan komut
```

### `changed_when` Kullanımı

```
Komut çalışır
       │
       ▼
changed_when koşulu değerlendirilir
       │
   ┌───┴───┐
 TRUE    FALSE
   │       │
   ▼       ▼
changed:  changed:
 true      false
```

```yaml
# RC'ye göre changed_when
- name: Migration çalıştır
  shell: python manage.py migrate
  register: migrate_output
  changed_when: migrate_output.rc != 0

# stdout içeriğine göre
- name: Servis durumunu kontrol et
  shell: systemctl is-active nginx
  register: nginx_status
  changed_when: nginx_status.stdout != "active"
  failed_when: false

# Hiçbir zaman changed sayma (sadece bilgi toplama)
- name: Disk kullanımını öğren
  command: df -h /
  register: disk_info
  changed_when: false

- name: Disk bilgisini göster
  debug:
    msg: "{{ disk_info.stdout }}"

# stdout_lines ile satır bazlı kontrol
- name: Paket listesini kontrol et
  shell: dpkg -l | grep mypackage
  register: pkg_check
  changed_when: pkg_check.stdout_lines | length == 0
  failed_when: false
```

### Birden Fazla Koşul

```yaml
# VE koşulu (her ikisi de true olmalı)
- name: Veritabanı migration
  shell: flask db upgrade
  register: db_migrate
  changed_when:
    - db_migrate.rc == 0
    - "'No migrations to apply' not in db_migrate.stdout"

# VEYA koşulu
- name: Servis yeniden başlatma kontrolü
  shell: service myapp status
  register: svc_status
  changed_when: >
    'stopped' in svc_status.stdout or
    'failed' in svc_status.stdout
```

---

## `failed_when` Kullanımı

Hangi durumda task'ın **başarısız** sayılacağını tanımlar.

```yaml
# Varsayılan: rc != 0 ise fail
# failed_when ile özelleştirme:

# Örnek 1: Belirli rc değerlerini görmezden gel
- name: Kullanıcının var olup olmadığını kontrol et
  command: id myuser
  register: user_check
  failed_when: user_check.rc > 1   # 0=var, 1=yok, 2+=gerçek hata
  changed_when: false

# Örnek 2: Stderr içeriğine göre fail
- name: Config dosyasını doğrula
  shell: nginx -t
  register: nginx_test
  failed_when: "'syntax error' in nginx_test.stderr"
  changed_when: false

# Örnek 3: Hiçbir zaman fail etme (ignore_errors alternatifi)
- name: Eski process'i durdur
  shell: pkill -f myapp
  failed_when: false
  changed_when: false

# Örnek 4: changed_when ve failed_when birlikte
- name: Backup scripti çalıştır
  shell: /usr/local/bin/backup.sh
  register: backup_result
  changed_when: backup_result.rc == 0
  failed_when:
    - backup_result.rc != 0
    - "'No space left' in backup_result.stderr"
```

---

## Gerçek Dünya Örnekleri

### Örnek 1: Java Uygulaması Kurulumu

```yaml
---
- name: Java Uygulama Sunucusu Kurulumu
  hosts: app_servers
  become: yes
  vars:
    java_version: "17"
    app_name: "myservice"
    app_version: "3.2.1"
    app_port: 8080

  tasks:
    # Idempotent: package modülü
    - name: Java'yı kur
      package:
        name: "openjdk-{{ java_version }}-jdk"
        state: present

    # Idempotent: user modülü
    - name: Servis kullanıcısı oluştur
      user:
        name: "{{ app_name }}"
        system: yes
        shell: /sbin/nologin
        home: "/opt/{{ app_name }}"
        create_home: yes

    # Idempotent: get_url (checksum ile)
    - name: JAR dosyasını indir
      get_url:
        url: "https://releases.example.com/{{ app_name }}-{{ app_version }}.jar"
        dest: "/opt/{{ app_name }}/app.jar"
        checksum: "sha256:abc123..."

    # Idempotent: template modülü
    - name: Systemd servis dosyasını oluştur
      template:
        src: app.service.j2
        dest: "/etc/systemd/system/{{ app_name }}.service"
      notify: restart service

    # Non-idempotent → creates ile idempotent
    - name: Veritabanı şemasını oluştur
      command: "/opt/{{ app_name }}/bin/db-init.sh"
      args:
        creates: "/opt/{{ app_name }}/.db_initialized"
      become_user: "{{ app_name }}"

    # Idempotent: systemd modülü
    - name: Servisi başlat ve enable et
      systemd:
        name: "{{ app_name }}"
        state: started
        enabled: yes
        daemon_reload: yes

  handlers:
    - name: restart service
      systemd:
        name: "{{ app_name }}"
        state: restarted
```

---

### Örnek 2: Docker Kurulumu ve Container Yönetimi

```yaml
---
- name: Docker Kurulumu
  hosts: docker_hosts
  become: yes

  tasks:
    # GPG key ekle (creates ile idempotent)
    - name: Docker GPG key indir
      shell: |
        curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
        gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
      args:
        creates: /usr/share/keyrings/docker-archive-keyring.gpg

    # Idempotent: apt_repository modülü
    - name: Docker repo ekle
      apt_repository:
        repo: "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu focal stable"
        state: present

    # Idempotent: apt modülü
    - name: Docker kur
      apt:
        name:
          - docker-ce
          - docker-ce-cli
          - containerd.io
        state: present
        update_cache: yes

    # Idempotent: user modülü
    - name: Deploy kullanıcısını docker grubuna ekle
      user:
        name: deploy
        groups: docker
        append: yes

    # register + changed_when örneği
    - name: Docker versiyonunu kontrol et
      command: docker --version
      register: docker_version
      changed_when: false   # Sadece bilgi toplama

    - name: Docker versiyon bilgisi
      debug:
        msg: "Kurulu Docker: {{ docker_version.stdout }}"
```

---

### Örnek 3: Nginx + SSL Kurulumu

```yaml
---
- name: Nginx SSL Kurulumu
  hosts: web_servers
  become: yes

  tasks:
    - name: Nginx ve Certbot kur
      apt:
        name:
          - nginx
          - certbot
          - python3-certbot-nginx
        state: present

    - name: Nginx site config yaz
      template:
        src: nginx-site.conf.j2
        dest: "/etc/nginx/sites-available/{{ domain }}"
      notify: reload nginx

    - name: Site'ı enable et
      file:
        src: "/etc/nginx/sites-available/{{ domain }}"
        dest: "/etc/nginx/sites-enabled/{{ domain }}"
        state: link
      notify: reload nginx

    # SSL sertifikası al (creates ile idempotent)
    - name: SSL sertifikası al
      command: >
        certbot --nginx -d {{ domain }}
        --non-interactive --agree-tos -m admin@{{ domain }}
      args:
        creates: "/etc/letsencrypt/live/{{ domain }}/fullchain.pem"

    # Config geçerliliği kontrol (failed_when örneği)
    - name: Nginx config'i test et
      command: nginx -t
      register: nginx_test
      changed_when: false
      failed_when: nginx_test.rc != 0

    - name: Nginx başlat
      systemd:
        name: nginx
        state: started
        enabled: yes

    # Certbot renewal cron (Idempotent: cron modülü)
    - name: Certbot renewal cron'u ayarla
      cron:
        name: "certbot renewal"
        minute: "0"
        hour: "12"
        job: "certbot renew --quiet"
        user: root

  handlers:
    - name: reload nginx
      systemd:
        name: nginx
        state: reloaded
```

---

### Örnek 4: Uygulama Sağlık Kontrolü

```yaml
---
- name: Uygulama Sağlık Kontrolü
  hosts: app_servers

  tasks:
    # HTTP endpoint kontrolü
    - name: API sağlık kontrolü
      uri:
        url: "http://localhost:8080/health"
        method: GET
        status_code: 200
      register: health_check
      failed_when: health_check.status != 200
      changed_when: false

    # Process kontrolü
    - name: Uygulama process'ini kontrol et
      shell: pgrep -f myapp
      register: process_check
      failed_when: false
      changed_when: false

    - name: Uygulama çalışmıyorsa başlat
      systemd:
        name: myapp
        state: started
      when: process_check.rc != 0

    # Log analizi
    - name: Son hataları kontrol et
      shell: grep -c ERROR /var/log/myapp/app.log
      register: error_count
      changed_when: false
      failed_when: error_count.stdout | int > 100

    - name: Hata sayısını raporla
      debug:
        msg: "Son log'da {{ error_count.stdout }} hata bulundu"
```

---

## İyi Pratikler ve Altın Kurallar

### 1. Öncelik Sırası

```
1. Idempotent Ansible modülü kullan          (En iyi)
2. Idempotent olmayan modül + creates/removes
3. Idempotent olmayan modül + register + when
4. Idempotent olmayan modül + ignore_errors  (Kaçın)
```

### 2. `changed_when` İçin Kullanım Kılavuzu

| Durum | Kullanım |
|---|---|
| Sadece bilgi toplama | `changed_when: false` |
| Her zaman değişiklik say | `changed_when: true` |
| RC'ye göre | `changed_when: result.rc != 0` |
| Çıktı içeriğine göre | `changed_when: "'keyword' in result.stdout"` |

### 3. Check Mode Desteği

```yaml
# Check mode ile uyumlu task
- name: Dosya var mı kontrol et
  stat:
    path: /etc/myapp.conf
  register: config_stat

- name: Config oluştur
  template:
    src: myapp.conf.j2
    dest: /etc/myapp.conf
  when: not config_stat.stat.exists
```

### 4. Kaçınılması Gereken Durumlar

```yaml
# ❌ KÖTÜ: Her çalıştırmada append eder
- shell: echo "export VAR=value" >> /etc/environment

# ✅ İYİ: Idempotent
- lineinfile:
    path: /etc/environment
    line: "VAR=value"
    regexp: '^VAR='

# ❌ KÖTÜ: Her çalıştırmada çalışır
- command: /opt/app/setup.sh

# ✅ İYİ: Bir kez çalışır
- command: /opt/app/setup.sh
  args:
    creates: /opt/app/.setup_complete

# ❌ KÖTÜ: Her zaman changed döner
- shell: systemctl status nginx
  register: result

# ✅ İYİ: Bilgi toplama, changed: false
- shell: systemctl status nginx
  register: result
  changed_when: false
```

### 5. Hızlı Başvuru Tablosu

| Sorun | Çözüm |
|---|---|
| `shell/command` her zaman changed döner | `changed_when: false` veya çıktıya göre koşul |
| Script iki kez çalışmamalı | `creates` ile flag dosyası |
| Hata kodunu görmezden gel | `failed_when: false` veya `ignore_errors: yes` |
| Koşula göre task çalıştır | `register` + `when` kombinasyonu |
| Task'ın ne yaptığını anlamak | `register` + `debug` modülü |

---

## 📚 Özet

```
Ansible Idempotency Piramidi:
                    ┌────────────────────────────────────┐
                    │  Idempotent Modüller               │ ← Her zaman tercih et
                    │  (file, copy, package, service)    │
                    ├────────────────────────────────────┤
                    │  command/shell + creates/removes   │ ← İkinci tercih
                    ├────────────────────────────────────┤
                    │  register + changed_when           │ ← Gerektiğinde
                    ├────────────────────────────────────┤
                    │  ignore_errors / failed_when:false │ ← Son çare
                    └────────────────────────────────────┘
```

> 💡 **Altın Kural:** Ansible playbook'unuz günde 100 kez çalışsa bile altyapınız bozulmamalıdır.  
> Bunu sağlamak tamamen doğru modül seçimi ve idempotency prensiplerine uymakla mümkündür.

---

*Hazırlayan: Ansible Idempotency Rehberi — Tüm hakları saklıdır.*


