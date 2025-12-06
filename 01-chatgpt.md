برای **ممیزی امنیتی Kubernetes بدون استفاده از ابزارهای آماده** (مثل kube-bench یا kubescape)، باید یک چک‌لیست مشخص را **به‌صورت دستی** بررسی کنید. در ادامه یک راهنمای کامل، عملی و مرحله‌به‌مرحله می‌دهم که بتوانید خودتان ممیزی امنیتی انجام دهید.

---

# ✅ **چک‌لیست کامل ممیزی امنیتی Kubernetes (بدون ابزار)**

## 🔹 1. بررسی امنیت Master Node (Control Plane)

### **۱) بررسی فایل‌های کانفیگ و سطح دسترسی**

مسیرهای مهم:

* `/etc/kubernetes/manifests/`
* `/var/lib/etcd/`
* `/etc/kubernetes/pki/`
* `/etc/kubernetes/admin.conf`

**باید بررسی کنید:**

* فایل‌ها فقط برای root خواندنی باشند

  ```bash
  ls -l /etc/kubernetes/
  ls -l /etc/kubernetes/pki/
  ```
* هیچ فایل Kubeconfig با permission باز وجود نداشته باشد.

---

### **۲) بررسی API Server**

فایل `/etc/kubernetes/manifests/kube-apiserver.yaml` را باز کنید و این موارد را چک کنید:

✔️ فعال بودن RBAC:

```
--authorization-mode=RBAC
```

✔️ غیرفعال بودن AlwaysAllow:

```
--authorization-mode=Node,RBAC
```

✔️ غیرفعال بودن insecure-port:

```
--insecure-port=0
```

✔️ TLS فعال باشد:

```
--tls-cert-file=
--tls-private-key-file=
```

✔️ Audit Logging فعال باشد:

```
--audit-log-path=/var/log/kube-apiserver-audit.log
```

---

### **۳) بررسی kube-controller-manager**

فایل `/etc/kubernetes/manifests/kube-controller-manager.yaml` را بررسی کنید:

✔️ فعال بودن محدودیت Tokenها:

```
--use-service-account-credentials=true
```

✔️ عدم وجود کلیدهای خارجی و اشتباه در فایل:

```
--root-ca-file=
--service-account-private-key-file=
```

---

### **۴) بررسی kube-scheduler**

در فایل `kube-scheduler.yaml`:

✔️ نباید هیچ **binding insecure** یا **hostNetwork** اضافه داشته باشد.

---

## 🔹 2. بررسی امنیت Worker Nodes

### **۱) بررسی kubelet**

فایل کانفیگ:

```
/var/lib/kubelet/config.yaml
```

چک کنید:

✔️ فعال بودن TLS:

```
tlsCertFile:
tlsPrivateKeyFile:
```

✔️ غیرفعال بودن anonymous auth:

```
anonymous:
  enabled: false
```

✔️ فعال بودن Authorization:

```
authorization:
  mode: Webhook
```

---

### **۲) بررسی اینکه kubelet روی insecure port کار نکند**

```bash
ps aux | grep kubelet
```

نباید شامل باشد:

```
--read-only-port=10255
```

---

## 🔹 3. بررسی امنیت شبکه (CNI)

✔️ بررسی کنید آیا NetworkPolicy فعال است؟

```bash
kubectl get networkpolicy -A
```

اگر صفر بود ⇒ امنیت شبکه **ضعیف** است.

✔️ بررسی Podها برای hostNetwork

```bash
kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{.spec.hostNetwork}{"\n"}{end}'
```

✔️ بررسی privileged pods:

```bash
kubectl get pods -A -o jsonpath='{.items[*].spec.containers[*].securityContext.privileged}'
```

---

# 🔹 4. بررسی RBAC و دسترسی‌ها

### **۱) کاربرهایی که Cluster Admin هستند**

```bash
kubectl get clusterrolebindings | grep cluster-admin
```

هر خروجی غیر لازم = ریسک امنیتی

---

### **۲) پیدا کردن ServiceAccountهای با دسترسی زیاد**

```bash
kubectl get rolebindings,clusterrolebindings -A
```

این‌ها باید بررسی شوند:

* آیا SA به صورت ناخواسته به نقش‌های admin متصل شده؟
* آیا هیچ سیستم خارجی توکن را خوانده است؟

---

### **۳) بررسی Secrets**

```bash
kubectl get secrets -A
```

✔️ بررسی کنید رمزها Base64 ساده‌اند
✔️ بررسی کنید Secretهای غیرضروری وجود دارد یا خیر

---

# 🔹 5. بررسی Pod Security

### **۱) بررسی Privileged بودن**

```bash
kubectl get pods -A -o json | grep -i privileged
```

### **۲) بررسی اینکه کانتینرها به Root اجرا نشوند**

```bash
kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.name}{" => "}{.spec.containers[*].securityContext.runAsNonRoot}{"\n"}{end}'
```

### **۳) بررسی capabilityهای خطرناک**

```bash
kubectl get pods -A -o json | grep -i CAP_
```

---

# 🔹 6. بررسی امنیت Docker / Container Runtime

✔️ بررسی ورژن containerd / docker

```bash
containerd --version
docker --version
```

✔️ بررسی مجوز `/etc/containerd/config.toml`
✔️ بررسی عدم اجرای کانتینرها با hostPID یا hostIPC

---

# 🔹 7. بررسی etcd

فایل:

```
/etc/kubernetes/manifests/etcd.yaml
```

✔️ TLS فعال باشد:

```
--cert-file=
--key-file=
--peer-cert-file=
--peer-key-file=
```

✔️ هیچ `--listen-metrics-urls=http://0.0.0.0` باز نباشد.

---

# 🔹 8. بررسی Logging و Auditing

✔️ Audit Logs فعال؟
✔️ kubelet logs فعال؟
✔️ etcd logs ذخیره می‌شود؟

فایل آپی‌سرور:

```
--audit-policy-file=
--audit-log-path=
```

---

# 🔹 9. بررسی موارد امنیتی در سطح سیستم‌عامل Nodes

✔️ بررسی iptables / nftables
✔️ بررسی kernel modules خطرناک
✔️ غیر فعال بودن swap
✔️ تنظیمات sysctl:

```bash
sysctl net.ipv4.ip_forward
sysctl net.ipv4.conf.all.accept_redirects
sysctl net.ipv4.conf.all.send_redirects
```

---

# 🔹 10. بررسی کانفیگ‌های مهم Cluster

✔️ بررسی ResourceQuota

```bash
kubectl get resourcequota -A
```

✔️ بررسی LimitRange

```bash
kubectl get limitrange -A
```

✔️ بررسی اینکه Podها بدون limit اجرا نشوند (رایج‌ترین مشکل امنیتی)

---

# 🎯 نتیجه:

اگر این موارد را دستی بررسی کنید، عملاً یک **ممیزی امنیتی حرفه‌ای، معادل kube-bench و kubescape** انجام داده‌اید.

اگر خواستی، می‌توانم برایت:

✅ یک چک‌لیست PDF تولید کنم
✅ اسکریپت Shell بدهم که همه این موارد را اتوماتیک چک کند
✅ بهت بگویم چطور گزارش رسمی Security Audit بنویسی

کدام را می‌خواهی؟
