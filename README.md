# Azure Function: Flex Consumption

Komutları Azure Portal'daki **Cloud Shell (Bash)** oturumunda çalıştırın. Cloud Shell oturumu zaten kimlik doğrulamış ve etkin aboneliği seçmiş olur.

## 1. Function App oluşturun

Flex Consumption destekleyen bölgeleri görüntüleyin ve kullanacağınız bölgeyi seçin:

```bash
az functionapp list-flexconsumption-locations --query "[].name" -o table
```

İsim çakışmalarını azaltmak için benzersiz kaynak adları üretip kaynak grubunu, depolamayı ve Node.js 22 kullanan Function App'i oluşturun. `LOCATION` değerini listelenen bir bölgeyle değiştirin.

```bash
SUFFIX=$(openssl rand -hex 3)
LOCATION="eastus"
RG="rg-func-http-$SUFFIX"
APP="fn-http-$SUFFIX"
STORAGE="stfunc$SUFFIX"

az group create --name "$RG" --location "$LOCATION"
az storage account create --name "$STORAGE" --resource-group "$RG" \
  --location "$LOCATION" --sku Standard_LRS --allow-blob-public-access false
az functionapp create --resource-group "$RG" --name "$APP" \
  --storage-account "$STORAGE" --flexconsumption-location "$LOCATION" \
  --runtime node --runtime-version 22 --https-only true
```

`az functionapp create` Application Insights kaynağını da varsayılan olarak oluşturur.

## 2. İletişim endpoint'ini oluşturup yayımlayın

`func` yüklü değilse Azure Functions Core Tools v4'ü kurun. Yeni JavaScript v4 projesi ve HTTP trigger oluşturun:

```bash
npm install -g azure-functions-core-tools@4 --unsafe-perm true
mkdir "functionTrigger-$SUFFIX" && cd "functionTrigger-$SUFFIX"
func init . --worker-runtime javascript --model V4
func new --template "HTTP trigger" --name HttpExample
```

`src/functions/HttpExample.js` içeriğini aşağıdakiyle değiştirin. GET tarayıcıda 200 döndürür; POST iletişim formu alanlarını doğrular. Bu örnek iletiyi kaydetmez veya e-posta göndermez.

```javascript
const { app } = require('@azure/functions');

app.http('HttpExample', {
  methods: ['GET', 'POST'],
  authLevel: 'anonymous',
  handler: async (request, context) => {
    if (request.method === 'GET') {
      return { status: 200, body: 'Contact endpoint is ready.' };
    }

    let contact;
    try {
      contact = await request.json();
    } catch {
      return { status: 400, jsonBody: { error: 'Invalid JSON body.' } };
    }

    const { name, email, message } = contact ?? {};
    const fields = [name, email, message];
    if (fields.some((value) => typeof value !== 'string' || !value.trim()) ||
        name.length > 100 || email.length > 254 || message.length > 5000) {
      return { status: 400, jsonBody: { error: 'Invalid contact fields.' } };
    }

    context.info('Contact form submission received.');
    return { status: 202, jsonBody: { message: 'Submission received.' } };
  },
});
```

Uygulamayı yayımlayın ve kendi web sitenizin origin'ine CORS izni verin:

```bash
func azure functionapp publish "$APP"
az functionapp cors add --resource-group "$RG" --name "$APP" \
  --allowed-origins "https://<SITE_DOMAIN>"
```

`<SITE_DOMAIN>` değerini sitenizin origin'iyle değiştirin (ör. `https://example.com`). CORS yalnızca tarayıcıların başka bir origin'den yaptığı çağrıları sınırlar; kimlik doğrulama veya kötüye kullanım koruması değildir. Endpoint anonim ve herkese açık olduğundan, gerçek ileti kaydı/e-posta gönderimi eklerken spam koruması ve sunucu tarafı doğrulamayı da değerlendirin.

## 3. Endpoint'i test edin ve logları inceleyin

Tarayıcıda `https://<APP_NAME>.azurewebsites.net/api/HttpExample` adresini açın. `<APP_NAME>` yerine Function App adını yazın; `Contact endpoint is ready.` yanıtı `200 OK` anlamına gelir. Application Insights loglarını Azure Portal'da Function App'in ilişkili Application Insights kaynağından inceleyin.
