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

## 2. HTTP-triggered function yayımlayın

`func` yüklü değilse Azure Functions Core Tools v4'ü kurun. Sonra yeni JavaScript v4 projesi ve HTTP trigger oluşturup uygulamaya yayımlayın:

```bash
npm install -g azure-functions-core-tools@4 --unsafe-perm true
mkdir "functionTrigger-$SUFFIX" && cd "functionTrigger-$SUFFIX"
func init . --worker-runtime javascript --model V4
func new --template "HTTP trigger" --name HttpExample
func azure functionapp publish "$APP"
```

## 3. Endpoint'i test edin

HTTP trigger varsayılan olarak anahtarsız çağrılabilir. Yayımladıktan sonra endpoint'e istek gönderin; yanıt `200 OK` olmalıdır.

`https://<APP_NAME>.azurewebsites.net/api/HttpExample?name=Azure` (`<APP_NAME>` yerine Function App adını yazın.)

Function App ve Application Insights Azure Portal'da oluşturulmuştur; logları Portal arayüzünden inceleyebilirsiniz.
