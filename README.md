# Azure Function: Flex Consumption

Bu kurulumun tamamını Azure Portal'daki **Cloud Shell (Bash)** içinde, aynı oturumda çalıştırın. Cloud Shell zaten Azure'da oturum açmıştır. Uygulama kodunu Portal editöründe değiştirmeyin: Flex Consumption deployment paketi read-only çalışır. Kodu Cloud Shell projesinde üretip yeniden yayımlayın.

## 1. Flex Consumption Function App oluşturun

Desteklenen bir bölge seçin:

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
printf 'Function App: %s\nResource Group: %s\n' "$APP" "$RG"
```

`az functionapp create` Application Insights kaynağını da varsayılan olarak oluşturur.

## 2. Projeyi Cloud Shell'de oluşturun

Cloud Shell'deki Core Tools sürümünü doğrulayın ve JavaScript v4 HTTP trigger projesini oluşturun:

```bash
func --version
mkdir "functionTrigger-$SUFFIX" && cd "functionTrigger-$SUFFIX"
func init . --worker-runtime javascript --model V4
func new --template "HTTP trigger" --name HttpExample
```

Portal code editor read-only olduğundan, function dosyasını doğrudan Cloud Shell terminalinde oluşturun:

```bash
cat > src/functions/HttpExample.js <<'EOF'
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
EOF
```

GET tarayıcıda `200 OK` döndürür. POST, iletişim formu alanlarını doğrular ve `202 Accepted` döndürür. Bu örnek iletiyi kaydetmez veya e-posta göndermez.

## 3. Yayımlayın ve CORS'u ayarlayın

Projeyi Azure'a yayımlayın. İletişim formunuz farklı bir web sitesindeyse, o sitenin origin'ine CORS izni verin:

```bash
func azure functionapp publish "$APP"
SITE_ORIGIN="https://your-site.example"
az functionapp cors add --resource-group "$RG" --name "$APP" \
  --allowed-origins "$SITE_ORIGIN"
```

`SITE_ORIGIN` değerini kendi sitenizin adresiyle değiştirin. CORS, key'siz `200` davranışını bozmaz; yalnızca tarayıcıdan farklı origin'lerle yapılan çağrıları sınırlar. CORS kimlik doğrulama değildir. Endpoint herkese açık ve anonimdir; örnek alanları doğrular ama iletiyi kaydetmez.

## 4. Endpoint'i açın ve logları inceleyin

Tarayıcıda şu adresi açın; `Contact endpoint is ready.` yanıtı `200 OK` demektir:

`https://<APP>.azurewebsites.net/api/HttpExample`

`<APP>` yerine 1. adımda yazdırılan Function App adını koyun. Application Insights loglarını Azure Portal'da, bu Function App'in ilişkili Application Insights kaynağının **Logs** bölümünden inceleyin.
