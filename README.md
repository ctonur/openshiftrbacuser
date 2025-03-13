# OpenShift Kullanıcı ve Yetkilendirme Yapılandırması

Bu proje, OpenShift ortamında 17 kullanıcı ve 15 namespace oluşturarak her kullanıcının sadece kendi namespace'ini görmesini sağlayan bir yapı içerir. Ayrıca, özel yetkilere sahip **"guvenlik"** ve **"readonly"** kullanıcıları da eklenmiştir.

## **📌 1. Kullanıcılar ve Yetkilendirme Senaryosu**
- **15 Kullanıcı (`user1` - `user15`)** → Her biri sadece kendi namespace'ini (`ns1` - `ns15`) görebilir ve yönetebilir.
- **1 Güvenlik Kullanıcısı (`guvenlik`)** → Tüm namespace'lerde yalnızca **NetworkPolicy yönetimi yapabilir, pod ve namespace listeleyebilir**.
- **1 Readonly Kullanıcısı (`readonly`)** → Sadece kendi namespace'inde **salt okunur yetkiye sahiptir, `oc logs` komutu ile logları görebilir**.

---

## **📌 2. Kullanıcıları ve Şifrelerini Oluşturma**
OpenShift’te kimlik doğrulama için `htpasswd` kullanıyoruz. Aşağıdaki komutları çalıştırarak 15 normal kullanıcıyı, `guvenlik` ve `readonly` kullanıcılarını oluşturabilirsiniz:

```bash
# 15 normal kullanıcı + 2 ekstra kullanıcı (guvenlik & readonly)
for i in {1..15}; do htpasswd -B -b users.htpasswd user$i 1234; done
htpasswd -B -b users.htpasswd guvenlik 1234
htpasswd -B -b users.htpasswd readonly 1234

# OpenShift'e secret olarak ekle
oc create secret generic htpasswd-secret --from-file=htpasswd=users.htpasswd -n openshift-config --dry-run=client -o yaml | oc apply -f -
```

Daha sonra, kimlik doğrulama sağlayıcısını OpenShift’e tanıtıyoruz:

```bash
oc patch oauth cluster --type=merge -p '{
  "spec": {
    "identityProviders": [{
      "name": "htpasswd_provider",
      "mappingMethod": "claim",
      "type": "HTPasswd",
      "htpasswd": {
        "fileData": { "name": "htpasswd-secret" }
      }
    }]
  }
}'
```

Podları yeniden başlatarak değişikliklerin aktif olmasını sağlıyoruz:

```bash
oc delete pod -n openshift-authentication --all
```

---

## **📌 3. Namespace’leri ve Kullanıcı Yetkilendirmelerini Oluşturma**

**Namespace’leri oluştur:**
```bash
for i in {1..15}; do oc new-project ns$i; done
```

**Her kullanıcıya sadece kendi namespace’inde admin yetkisi verme:**
```bash
for i in {1..15}; do
  oc adm policy add-role-to-user admin user$i -n ns$i
done
```

---

## **📌 4. Guvenlik Kullanıcısına Özel Yetkilendirme**
`guvenlik` kullanıcısına tüm namespace’lerde **sadece NetworkPolicy yönetimi yetkisi** veriyoruz:

```bash
oc create clusterrole guvenlik-role --verb=list,get,update --resource=networkpolicies.networking.k8s.io
oc create clusterrole guvenlik-view --verb=list --resource=pods,namespaces
```

Rolü `guvenlik` kullanıcısına veriyoruz:
```bash
oc adm policy add-cluster-role-to-user guvenlik-role guvenlik
oc adm policy add-cluster-role-to-user guvenlik-view guvenlik
```

---

## **📌 5. Readonly Kullanıcısına Sadece Okuma Yetkisi Verme**
`readonly` kullanıcısı sadece kendi namespace'inde **view (salt okunur) yetkisine** sahip olacak:

```bash
oc adm policy add-role-to-user view readonly -n ns1
```

> **Not:** `readonly` kullanıcısının namespace’ini değiştirmek için `-n ns1` kısmını değiştir.

---

## **📌 6. Kullanıcıların OpenShift’e Giriş Yapması**
Her kullanıcı şu komutları kullanarak OpenShift’e giriş yapabilir:
```bash
oc login -u user1 -p 1234 --server=<API_SERVER>
oc project ns1  # Kendi namespace'ine geç
```

`readonly` kullanıcısı logları görmek için şu komutu çalıştırabilir:
```bash
oc logs -n ns1 <POD_NAME>
```

---

## **📌 7. Kullanıcıyı veya Yetkisini Silmek**
Bir kullanıcıyı OpenShift’ten tamamen silmek için:
```bash
oc delete user user1
oc delete identity htpasswd_provider:user1
```

Bir kullanıcının yetkisini iptal etmek için:
```bash
oc adm policy remove-role-from-user admin user1 -n ns1
```

`htpasswd` dosyasından bir kullanıcıyı kaldırmak için:
```bash
htpasswd -D users.htpasswd user1
```
Ardından, güncellenmiş dosyayı OpenShift’e tekrar yükleyin:
```bash
oc create secret generic htpasswd-secret --from-file=htpasswd=users.htpasswd -n openshift-config --dry-run=client -o yaml | oc apply -f -
```

---

## **📌 8. Test ve Kontroller**
Kullanıcı girişlerini ve yetkilerini doğrulamak için şu komutları kullanabilirsiniz:
```bash
oc whoami  # Giriş yapan kullanıcıyı gösterir
oc projects  # Kullanıcının erişebildiği namespace'leri listeler
oc get pods -n ns1  # Kullanıcının eriştiği namespace’de podları listeler
```

# OpenShift `oc` Komutları ile Kullanıcı ve Yetki Yönetimi

Bu belge, OpenShift ortamında kullanıcıları, rolleri ve yetkilendirmeleri yönetmek için kullanılan `oc` komutlarını açıklar.

---

## **📌 1. Kullanıcıları ve Rolleri Listeleme**

### **Mevcut Kullanıcıları Görüntüleme**
```bash
# OpenShift’te tanımlı kullanıcıları listele
oc get users

# Kullanıcıların kimlik bilgilerini listele
oc get identities

# Belirli bir kullanıcının detaylarını görmek için
oc get user <KULLANICI_ADI> -o yaml
```

### **Mevcut Rolleri Görüntüleme**
```bash
# Tüm rollerin listesini görmek için
oc get roles --all-namespaces

# Belirli bir namespace içindeki rolleri listeleme
oc get roles -n <NAMESPACE>

# Cluster-wide (küresel) rollerin listesini görmek için
oc get clusterroles
```

---

## **📌 2. ClusterRole ve RoleBinding’leri Görüntüleme**

### **Mevcut ClusterRole’leri Listeleme**
```bash
# Tüm ClusterRole’leri görmek için
oc get clusterroles

# Bir ClusterRole'in detaylarını görmek için
oc describe clusterrole <CLUSTER_ROLE_ADI>
```

### **RoleBinding’leri Listeleme**
```bash
# Bir namespace içindeki RoleBinding’leri listele
oc get rolebindings -n <NAMESPACE>

# RoleBinding detaylarını görmek için
oc describe rolebinding <ROLEBINDING_ADI> -n <NAMESPACE>

# Tüm cluster genelindeki RoleBinding’leri görmek için
oc get clusterrolebindings
```

---

## **📌 3. OpenShift’te Geçici Kullanıcı Girişi (Token ile)**

### **Token ile Geçici Giriş Yapma**
```bash
oc login --token=<TOKEN> --server=<API_SERVER>
```

### **Token’i Alma**
```bash
oc whoami -t
```
Bu yöntemle cluster’a giriş yaparsın, ancak terminali kapattığında veya oturumu değiştirdiğinde yetki kaybolur.

---

## **📌 4. Kullanıcıları Zorla Oturumu Kapatma (`oc adm prune auth`)**

### **Tüm Kullanıcıları Zorla Oturumu Kapatma**
```bash
oc adm prune auth
```

### **Bu Komut Ne Yapar?**
- Kullanıcıların mevcut kimlik doğrulama bilgilerini temizler.
- Kullanıcıları sistemden çıkış yaptırır.
- Önbelleğe alınmış eski yetkileri temizler.
- OpenShift API’ye tekrar giriş yapmalarını gerektirir.

### **Belirli Bir Kullanıcıyı Sistemden Çıkarma**
```bash
oc delete user <KULLANICI_ADI>
oc delete identity htpasswd_provider:<KULLANICI_ADI>
```

---

## **📌 5. Test ve Kontroller**
Aşağıdaki komutlar ile yetkilendirmeleri test edebilirsiniz:

```bash
oc whoami  # Giriş yapan kullanıcıyı gösterir
oc projects  # Kullanıcının erişebildiği namespace'leri listeler
oc get pods -n <NAMESPACE>  # Kullanıcının eriştiği namespace’de podları listeler
```


