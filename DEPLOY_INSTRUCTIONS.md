# دليل رفع التطبيق على VPS (Ubuntu/Debian)

## 1. إعداد الخادم (Server Setup)
قم بتحديث النظام وتثبيت المتطلبات الأساسية:
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl dirmngr apt-transport-https lsb-release ca-certificates nginx mysql-server
```

## 2. إعداد قاعدة البيانات (MySQL)
قم بتأمين وتكوين MySQL وإنشاء قاعدة البيانات:
```bash
sudo mysql_secure_installation
```
ثم ادخل إلى MySQL:
```bash
sudo mysql -u root -p
```
نفذ الأوامر التالية:
```sql
CREATE DATABASE employee_cards_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'emp_user'@'localhost' IDENTIFIED BY 'StrongPassword123!';
GRANT ALL PRIVILEGES ON employee_cards_db.* TO 'emp_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

## 3. تثبيت Node.js و PM2
```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-install -y nodejs
sudo npm install -g pm2
```

## 4. رفع وتشغيل الباك إند (Backend) وإنشاء الجداول
1. انتقل لمجلد `backend`:
```bash
cd /path/to/project/backend
npm install
```
2. قم بتعديل أو إنشاء ملف `.env`:
```env
PORT=5000
DB_HOST=localhost
DB_USER=emp_user
DB_PASSWORD=StrongPassword123!
DB_NAME=employee_cards_db
JWT_SECRET=your_super_secret_key
NODE_ENV=production
ALLOWED_ORIGINS=https://nda.id.ly
```
3. قم بتهيئة قاعدة البيانات وإنشاء الجداول (تلقائياً يقوم بإنشاء حساب المسؤول الافتراضي):
```bash
node init_db.js
```
ملاحظة: تأكد من حفظ بيانات حساب المسؤول (Admin) التي ستظهر في الشاشة بعد تنفيذ الأمر.

4. ابدأ التشغيل باستخدام PM2:
```bash
pm2 start ecosystem.config.js --env production
pm2 save
pm2 startup
```

## 5. تجهيز وتشغيل الواجهة (Frontend)
1. في مجلد الواجهة الأساسي (المشروع):
```bash
cd /path/to/project
npm install
```
2. قم بإعداد رابط API للإنتاج بإنشاء ملف `.env.production` (اختياري لكن مفضل لخوادم الإنتاج):
```env
VITE_API_URL=https://nda.id.ly/api
```
أو تأكد من تعديل `VITE_API_URL` في ملف `.env` الأساسي.

ثم قم ببناء المشروع:
```bash
npm run build
```
3. سيتم إنشاء مجلد `dist`، سنقوم بربطه في Nginx.

## 6. إعداد Nginx (Reverse Proxy)
قم بإنشاء ملف تكوين لـ Nginx:
```bash
sudo nano /etc/nginx/sites-available/employee-cards
```
أضف التكوين التالي (مع تغيير النطاقات والمسارات):
```nginx
server {
    listen 80;
    server_name nda.id.ly; # ضع النطاق هنا

    # مسار الواجهة Frontend
    root /path/to/project/dist;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    # مسار الباك إند Backend API
    location /api/ {
        proxy_pass http://localhost:5000/api/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }

    # مسار الصور المرفوعة Uploads
    location /uploads/ {
        proxy_pass http://localhost:5000/uploads/;
        proxy_set_header Host $host;
    }
}
```

ثم قم بتفعيله:
```bash
sudo ln -s /etc/nginx/sites-available/employee-cards /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

## 7. شهادة الأمان SSL (اختياري لكن مستحسن جداً)
استخدم Certbot للحصول على شهادة مجانية:
```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d nda.id.ly
```
