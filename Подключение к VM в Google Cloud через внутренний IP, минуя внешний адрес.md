Dыполняем подготовку в терминале (PowerShell или CMD):

1️⃣ Включаем IAP API
```bash

gcloud services enable iap.googleapis.com
```


2️⃣ Проверяем, что у вас есть права на подключение через IAP
```bash

gcloud projects get-iam-policy ascendant-choir-468617-i0 --flatten="bindings[].members" --format="table(bindings.role,bindings.members)" --filter="iap"
```


В выводе должны быть роли вроде:

- `roles/iap.tunnelResourceAccessor` (или выше, например, `roles/editor` или `roles/owner`)
    и` ваш аккаунт `vnovo.biz@gmail.com`

3️⃣ Если роли нет — добавляем
```bash

gcloud projects add-iam-policy-binding ascendant-choir-468617-i0 \
  --member="user:vnovo.biz@gmail.com" \
  --role="roles/iap.tunnelResourceAccessor"
```
4️⃣ После этого — запускаем файл *`smart_iap_ssh.bat`*

Он создаст ключ (если нет) и подключит к VM через внутренний IP, минуя внешний адрес.