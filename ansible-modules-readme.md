# 📦 Ansible Modülleri: Idempotent vs Non-Idempotent

> Ansible'da modül seçimi, güvenilir ve tekrarlanabilir otomasyon için kritik öneme sahiptir.

---

## ✅ Idempotent Modüller

Her çalıştırıldığında aynı sonucu verir. Sistem zaten istenen durumda ise herhangi bir değişiklik yapılmaz.

| Modül | Açıklama |
|---|---|
| `copy` | Dosya kopyalar; aynıysa dokunmaz |
| `template` | Jinja2 şablonunu render edip yazar |
| `file` | Dosya/dizin/semlink oluşturur veya siler |
| `user` | Kullanıcı oluşturur veya günceller |
| `group` | Grup yönetimi |
| `package` / `apt` / `yum` / `dnf` | Paket yükler; zaten yüklüyse geçer |
| `service` / `systemd` | Servis başlatır/durdurur; zaten o durumdaysa geçer |
| `lineinfile` | Dosyaya satır ekler; zaten varsa eklemez |
| `blockinfile` | Dosyaya blok ekler; zaten varsa eklemez |
| `cron` | Cron job yönetimi |
| `git` | Repo klonlar veya günceller |
| `authorized_key` | SSH key yönetimi |
| `mount` | Dosya sistemi bağlama |
| `sysctl` | Kernel parametre yönetimi |
| `firewalld` / `ufw` | Güvenlik duvarı kural yönetimi |

---

## ⚠️ Non-Idempotent Modüller

Her çalıştırıldığında bir eylem gerçekleştirir. Tekrar çalıştırıldığında beklenmedik yan etkiler oluşabilir.

| Modül | Neden Non-Idempotent? |
|---|---|
| `command` | Her seferinde komutu çalıştırır |
| `shell` | Her seferinde shell komutunu çalıştırır |
| `raw` | Ham SSH komutu, kontrol mekanizması yok |
| `script` | Yerel scripti her seferinde çalıştırır |
| `uri` (POST/DELETE) | HTTP isteği atar; tekrar atarsa etkisi değişebilir |

---

## 🛠️ Non-Idempotent Modülleri Idempotent Yapmak

`command` ve `shell` modüllerini idempotent hale getirmek için aşağıdaki yöntemler kullanılabilir:

### `creates` parametresi ile
```yaml
- name: Sadece dosya yoksa çalıştır
  command: /usr/bin/setup.sh
  args:
    creates: /etc/setup.done
```

### `register` + `when` ile koşullu çalıştırma
```yaml
- name: Durum kontrolü
  command: systemctl is-active myservice
  register: svc_status
  ignore_errors: true

- name: Sadece durmazsa çalıştır
  command: /usr/bin/dosomething.sh
  when: svc_status.rc != 0
```

### `changed_when` ile değişiklik kontrolü
```yaml
- name: Sonuca göre changed durumu belirle
  shell: some_command
  register: result
  changed_when: "'updated' in result.stdout"
```

---

## 💡 Özet

```
Idempotent    →  Desired state tanımlar  →  Ansible farkı hesaplar  →  Güvenli ✅
Non-Idempotent →  Eylem tanımlar         →  Her seferinde çalışır   →  Dikkatli kullan ⚠️
```

Ansible'ın gücü **idempotent modülleri** tercih etmekten gelir.  
`shell` / `command` kullanmak zorundaysanız mutlaka `when`, `creates` veya `changed_when` ile koruyun.

---

## 📚 Kaynaklar

- [Ansible Resmi Dokümantasyon](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/)
- [Ansible Best Practices](https://docs.ansible.com/ansible/latest/tips_tricks/ansible_tips_tricks.html)
