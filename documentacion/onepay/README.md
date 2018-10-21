# Onepay

<div class="pos-title-nav">
  <div tbk-link='/referencia/onepay' tbk-link-name='Referencia Api'></div>
  <div tbk-link='/plugin/onepay' tbk-link-name='Plugins'></div>
</div>


## Conceptos y SDKs

El flujo de Onepay varía ligeramente según el *canal* mediante el cual
se enlaza la transacción con la app Onepay en el teléfono del usuario.
Los canales son:

- `WEB`: Web desktop. En este caso aparece un QR en la pantalla del comercio que
le permite al usuario escanearlo con la app Onepay y confirmar la transacción.
El comercio es notificado cuando el usuario ha completado la transacción.

- `MOBILE`: Web móvil. Aquí el navegador móvil abre la app Onepay, donde
el usuario confirma directamente la transacción. Luego la app Onepay abre *una
nueva pestaña del navegador del sistema* para notificar al comercio que se ha
completado la transacción.

- `APP`: Integración app-to-app. Aquí el comercio cuenta con una app móvil que
abre la app Onepay cuando el usuario desea pagar usando este medio de pago. Una
vez se completa la transacción, la app Onepay invoca nuevamente la app del
comercio.

Para los dos primeros canales (`WEB` y `MOBILE`) se ofrece un [SDK Javascript](https://github.com/TransbankDevelopers/transbank-sdk-js-onepay#integraci%C3%B3n-checkout)
con dos modalidades de integración:

- **Checkout**: Integración recomendada que maneja toda la comunicación con Onepay
e informa al usuario de los pasos a seguir a través de un diálogo "modal". En
esta modalidad el comercio indica dos urls que conectan el frontend del comercio
con el backend. La primera url es la encargada de crear la transacción. La
segunda url es la que toma el control después de que el usuario completa el
flujo en la app. **Checkout se encarga de hacer invisibles las diferencias entre
el canal `WEB` y `MOBILE`.**

- **QR Directo**: La versión "hágalo usted mismo" para usuarios avanzados.
Provee más flexibilidad pero es más compleja y requiere más trabajo la
integración. Se le entregan al comercio las herramientas para dibujar el
código QR y para "escuchar" los eventos que van ocurriendo en la app del
usuario. También se incluye una forma de abrir la app de Onepay en caso de
canal `MOBILE`, pero toda la lógica debe ser manejada por el comercio.

Para el canal `APP` se ofrecen SDKs [iOS](https://github.com/TransbankDevelopers/transbank-sdk-ios-onepay)
y [Android](https://github.com/TransbankDevelopers/transbank-sdk-android-onepay)

Finalmente, para las labores del backend, se ofrecen SDKs para [Java](https://github.com/TransbankDevelopers/transbank-sdk-java), [.NET](https://github.com/TransbankDevelopers/transbank-sdk-dotnet) y [PHP](https://github.com/TransbankDevelopers/transbank-sdk-php). (Python y Ruby
prontamente). Estos SDKs se utilizan en todos los casos, independiente del
canal y de la modalidad de integración web.

## Integración Checkout


### 1. Front-end: Configuración.

Para realizar la integración checkout debes utilizar [nuestro SDK Javascript](https://github.com/TransbankDevelopers/transbank-sdk-js-onepay) siguiendo las
[instrucciones de instalación](https://github.com/TransbankDevelopers/transbank-sdk-js-onepay#instalaci%C3%B3n).

Luego en tu frontend debes invocar a `Onepay.checkout` de la siguiente forma:

```javascript
Onepay.checkout({
  endpoint: './transaction-create',
  commerceLogo: 'https://tu-url.com/images/icons/logo-01.png',
  callbackUrl: './onepay-result'
});
```

Esto abrirá un modal que se encargará de toda la interacción con el usuario.

A continuación una descripción de los parámetros recibidos por esta función:

- `endpoint` : corresponde a la URL de tu backend que tiene la lógica de crear
la transacción usando alguno de nuestros SDK disponibles o invocando
directamente al API de Onepay.

El SDK enviara el parámetro `channel` a tu endpoint, cuyo valor podría ser
`"WEB"` o `"MOBILE"`. Debes asegurarte de capturar este parámetro para poder
enviar el channel adecuado al API de Onepay.

Se espera que el `endpoint` retorne un JSON como el del siguiente ejemplo:

```json
{
 "externalUniqueNumber":"38bab443-c55b-4d4e-86fa-8b9f4a2d2d13",
 "amount":88000,
 "qrCodeAsBase64":"QRBASE64STRING",
 "issuedAt":1534216134,
 "occ":"1808534370011282",
 "ott":89435749
}
```

En el paso 2 más abajo podrás ver más información sobre este `endpoint`.

- `commerceLogo`: Corresponde a la URL full del logo de comercio que se mostrará
  en el modal. Como el modal reside en un dominio externo, no puede ser una URL
  relativa (a diferencia de los otros parámetros). El logo se redimensionará a
  125 pixeles de ancho y la altura se calcula automáticamente para mantener las
  proporciones de la imagen.


- `callbackUrl` : URL que se invocará desde el SDK una vez que la transacción
ha sido autorizada por el comercio. En este callback el comercio debe hacer la
confirmación de la transacción, para lo cual dispone de 30 segundos desde que
la transacción se autorizó, de lo contrario esta sera automáticamente reversada.

En el paso 3 más abajo podrás ver más sobre cómo se invoca este _callback_.

<aside class="notice">
Tip: En desarrollo puedes comenzar tus URLs de callback con // en lugar de
http:// o https:// para evitar problemas al mezclar páginas seguras con
inseguras. Pero en producción siempre debieras usar https.
</aside>

### 2. Back-end: Crear una Transacción

Como puedes imaginar, para completar la integración necesitarás programar el
código que maneja las llamadas en las URLs indicadas por `endpoint` y
`callbackUrl`.

Primero que nada, deberás configurar también el `callbackUrl` en el backend.
La razón detrás de esto es que si el canal es `MOBILE` no es el componente
javascript quien invocará el _callback_, sino que la propia app Onepay que
reside en el teléfono del usuario. La configuración del callback en el backend
le permite a la app Onepay saber donde redireccionar en ese caso:

<div class="language-simple" data-multiple-language></div>

```java
import cl.transbank.onepay.Onepay;


// URL de retorno para canal MOBILE (web móvil). También será usada en canal WEB
// si integras la modalidad checkout del SDK javascript.
Onepay.setCallbackUrl("https://www.misitioweb.com/onepay-result");
```

```php
use Transbank\Onepay\OnepayBase;

OnepayBase::setCallbackUrl('https://www.misitioweb.com/onepay-result');
```

```csharp
using Transbank.Onepay;

Onepay.CallbackUrl = "https://www.misitioweb.com/onepay-result";
```


Con eso estás preparado para crear una transacción. Para esto se debe crear en
primera instancia un objeto `ShoppingCart` que se debe llenar un `Item` (o
varios):


<div class="language-simple" data-multiple-language></div>

```java
import cl.transbank.onepay.model.*;

//...

ShoppingCart cart = new ShoppingCart();
cart.add(
    new Item()
        .setDescription("Zapatos")
        .setQuantity(1)
        .setAmount(15000)
        .setAdditionalData(null)
        .setExpire(-1));
```

```php
use Transbank\Onepay\ShoppingCart;
use Transbank\Onepay\Item;
use Transbank\Onepay\Transaction;

$cart = new ShoppingCart();
$cart->add(new Item('Zapatos', 1, 15000));
```

```csharp
using Transbank.Onepay:
using Transbank.Onepay.Model:

//...

ShoppingCart cart = new ShoppingCart();
cart.Add(new Item(
    description: "Zapatos",
    quantity: 1,
    amount: 15000,
    additionalData: null,
    expire: -1));
```

El monto en el carro de compras debe ser positivo, en caso contrario se lanzará una excepción.

Luego que el carro de compras contiene todos los ítems, se crea la transacción:


<div class="language-simple" data-multiple-language></div>

```java
import cl.transbank.onepay.model.*;

// ...

Onepay.Channel channel = Onepay.Channel.valueOf(request.getParameter("channel"));
TransactionCreateResponse response = Transaction.create(cart, channel);
```

```php
$channel = $request->input("channel");
$response = Transaction::create($cart, $channel);

```

```csharp
using Transbank.Onepay;
using Transbank.Onepay.Model:

// ...
ChannelType channelType = ChannelType.Parse(channel);
var response = Transaction.Create(cart, channelType);
```

El segundo parámetro en el ejemplo corresponde al `channel` y debes capturar el
valor que el frontend envió cuando invoco al endpoint que está creando la transacción para
fijar el valor correcto ("WEB" si se trata de web desktop, "MOBILE" si se trata
de web móvil).

El resultado entregado contiene la confirmación de la creación de la transacción, en la forma de un objeto `TransactionCreateResponse`.

```json
"occ": "1807983490979289",
"ott": 64181789,
"signature": "USrtuoyAU3l5qeG3Gm2fnxKRs++jQaf1wc8lwA6EZ2o=",
"externalUniqueNumber": "f506a955-800c-4185-8818-4ef9fca97aae",
"issuedAt": 1532103896,
"qrCodeAsBase64": "QRBASE64STRING"
```

El `externalUniqueNumber` corresponde a un UUID generado por SDK que identifica
la transacción en el lado del comercio. Opcionalmente, puedes [especificar tus
propios external unique numbers](/referencia/onepay#especificar-tus-propios-external-unique-numbers).

En el caso que no se pueda completar la transacción o `responseCode` en la
respuesta del API sea distinto de `ok` se lanzará una excepción.

Con eso ya tienes la información suficiente para responder el JSON que espera
la modalidad checkout, pues sólo debes eliminar `signature` y agregar `amount`
desde el `ShoppingCart` que construiste anteriormente. Con eso retornas algo
como esto:

```json
{
"occ": "1807983490979289",
"ott": 64181789,
"externalUniqueNumber": "f506a955-800c-4185-8818-4ef9fca97aae",
"issuedAt": 1532103896,
"qrCodeAsBase64": "QRBASE64STRING",
"amount":88000
}
```

Y en este punto el SDK frontend toma el control de la coordinación.  Si el canal
es `WEB` se le mostrará un "modal" al usuario donde podrá escanear el código QR
y también recibirá las instrucciones de cada paso a realizar para autorizar el
pago. Si el canal es `MOBILE` el SDK javascript automáticamente abrirá la app
Onepay en el mismo teléfono del usuario.

### 3. Back-end: Confirmar la Transacción

Eventualemnte la transacción llegará a término y el control retornará al backend
mediante el metodo `GET` en la url indicada en el `callbackUrl` en el paso 1
(frontend) y 2 (backend). Es muy importante que pongas la misma URL en ambos
pasos. De esta forma sólo tienes que escribir un único código que maneje la
confirmación de la transacción independiente si el canal es `WEB` o `MOBILE`.

Para el canal `WEB` (desktop) en la integración checkout el resultado de éxito o
fracaso le aparecerá comunicado al usuario dentro del diálogo modal, para luego
redirigir al usuario al _callback_ espeficado en el paso 1. Para el canal
`MOBILE` la app de Onepay abrirá una **nueva pestaña** en el navegador móvil del
usuario y en esa pestaña se invocará la url de _callback_ especificado en el
paso 2.

En este callback el comercio debe hacer la confirmación de la transacción, para
lo cual dispone de 30 segundos desde que la transacción se autorizó. De lo
contrario esta sera automáticamente reversada.

El callback será invocado vía `GET` e irán los parametros `occ` y
`externalUniqueNumber` con los cuales podrás invocar la confirmación de la
transacción desde tu backend. Adicionalmente se envía el parámetro `status`,
que puede tomar los siguientes valores:

- `PRE_AUTHORIZED`: El estado de éxito. Debes confirmar la transacción para
  completar el flujo.
- `CANCELLED_BY_USER`: El tarjetahabiente decidió abortar la autorización.
- `REJECTED`: La transacción fue rechazada.
- `REVERSED`: No se pudo confirmar el pago.
- `REVERSE_NOT_COMPLETED`: Se intentó reversar (ver estado anterior) pero falló
  por alguna razón interna.

Si se recibe cualquier otro valor, el comercio debe asumir que ocurrió un error.

Finalmente, se debe confirmar la transacción:

<div class="language-simple" data-multiple-language></div>

```java
import cl.transbank.onepay.model.*;

String status = request.getParameter("status");
String externalUniqueNumber = request.getParameter("externalUniqueNumber");
String occ = request.getParameter("occ");

if (null != status && status.equalsIgnoreCase("PRE_AUTHORIZED")) {
    try { 
        TransactionCommitResponse response =
            Transaction.commit(occ, externalUniqueNumber);
        // Procesar response
    } catch (TransbankException e) {
        // Error al confirmar la transaccion
    }
} else {
    // Mostrar página de error
}
```

```php
use Transbank\Onepay\Transaction;

$status = $request->input("status");
$externalUniqueNumber = $request->input("externalUniqueNumber");
$occ = $request->input("occ");

if (!empty($status) && strcasecmp($status,"PRE_AUTHORIZED") == 0) {
    try { 
        $response = Transaction::commit($occ, $externalUniqueNumber);
        // Procesar $response
    } catch (Exception $e) {
        // Error al confirmar la transaccion
    }
} else {
    // Mostrar página de error
}
```

```csharp
using Transbank.Onepay;

 if (null != status && status.Equals("PRE_AUTHORIZED", StringComparison.InvariantCultureIgnoreCase))
 {
    try
    { 
        var response = Transaction.Commit(occ, externalUniqueNumber);
        // Procesar response
    }
    catch (TransbankException e)
    {
        // Error al confirmar la transaccion
    }
}
else
{
    // Mostrar página de error
}
```

El resultado obtenido en `response` tiene una forma como la de este ejemplo:

```json
"occ": "1807983490979289",
"authorizationCode": "623245",
"issuedAt": 1532104549,
"signature": "FfY4Ab89rC8rEf0qnpGcd0L/0mcm8SpzcWhJJMbUBK0=",
"amount": 27500,
"transactionDesc": "Venta Normal: Sin cuotas",
"installmentsAmount": 27500,
"installmentsNumber": 1,
"buyOrder": "20180720122456123"
```

Para completar el flujo, solo queda que le des la noticia al usuario de que el
pago ha sido exitoso y le muestres un comprobante de ser necesario.


<aside class="notice">
Como el _callback_ final está implementado usando el método
HTTP GET, debe ser idempotente: Debe funcionar correctamente aunque sea
ejecutado repetidas veces.
</aside>

## Integración en App Móvil

Si tu comercio posee una app móvil, entonces debes integrar también los SDKs
móviles. Comienza revisando la manera de [instalar el SDK
Android](https://github.com/TransbankDevelopers/transbank-sdk-android-onepay#instalaci%C3%B3n)
y [el SDK
iOS](https://github.com/TransbankDevelopers/transbank-sdk-ios-onepay#instalaci%C3%B3n).

A diferencia de la modalidad Checkout, en la integración con tu app tú debes
encargarte directamente de la comunicación entre tu app y tu backend.

<aside class="warning">
Importante: Las transacciones deben ser siempre creadas y confirmadas por
el backend, pues es en tus servidores donde puedes validar que se estén
generando transacciones por los montos correctos (y no seas víctima de un
ataque malicioso).
</aside>

Pero como ya tienes creado el código que maneja la transacción, lo que te falta
por hacer eso sólo conectar las partes.

Primero tienes en el backend que configurar el equivalente al `callbackUrl` pero
para apps. Se trata del `appScheme`, que le dice a la app de Onepay cómo
entregarle de vuelta el control a tu propia aplicación:

<div class="language-simple" data-multiple-language></div>

```java
import cl.transbank.onepay.Onepay;

Onepay.setAppScheme("mi-app://mi-app/onepay-result");
```

```php
use Transbank\Onepay\OnepayBase;

OnepayBase::setAppScheme("mi-app://mi-app/onepay-result");
```

```csharp
using Transbank.Onepay:

Onepay.AppScheme = "mi-app://mi-app/onepay-result";
```

Luego puedes invocar al mismo endpoint que construiste para Checkout, pero ahora
lo llamas desde tu app pasando `APP` en el parámetro `channel` (via POST).

Con eso obtendrás en tu app los datos de la nueva transacción. Usando el `occ`
puedes usar nuestro SDK para invocar la app de Onepay:

```swift
import OnePaySDK
import os.log
//...

let onepay = OnePay();
onepay.initPayment("aqui-va-la-occ",
    callback: {(statusCode, description) in
        switch statusCode {
        case OnePayState.occInvalid:
            // Algo anda mal con el occ que obtuviste desde el backend
            // Debes reintentar obtener el occ o abortar
            os_log("OCC inválida")
        case OnePayState.notInstalled:
            // Onepay no está instalado.
            // Debes abortar o pedir al usuario instalar Onepay (y luego reintentar initPayment)
            os_log("Onepay no instalado")
        }
    }
)
```

En Android:

```java
import cl.ionix.tbk_ewallet_sdk_android.Error;
import cl.ionix.tbk_ewallet_sdk_android.OnePay;
import cl.ionix.tbk_ewallet_sdk_android.callback.OnePayCallback;
//...

OnePay onepay = new OnePay(this);
onepay.initPayment("occ", new OnePayCallback() {
    @Override
    public void failure(Error error, String s) {
        switch (error) {
            case INVALID_OCC:
                // Algo anda mal con el occ que obtuviste desde el backend
                // Debes reintentar obtener el occ o abortar
                break;
            case ONE_PAY_NOT_INSTALLED:
                // Onepay no está instalado.
                // Debes abortar o pedir al usuario instalar Onepay (y luego reintentar initPayment)
                break;
            default:
                Log.e(TAG, "Error inesperado al iniciar pago Onepay " + error.toString() + ":" + s);
                // Aborta o reintenta
        }
    }
});
```

Si todo funciona OK, el control pasará a la app Onepay donde el usuario podrá autorizar la transacción.

Una vez el usuario realice el pago, el control volverá a tu app usando el
`appScheme` que configuraste anteriormente.

Para eso, en iOS debes agregar una entrada a `Info.plist` para majar el scheme
indicado. Por ejemplo, si tu appScheme fuera mi-app://mi-app/onepay-result,
agregarías lo siguiente:

```xml
	<key>CFBundleURLTypes</key>
	<array>
		<dict>
			<key>CFBundleURLSchemes</key>
			<array>
				<string>mi-app</string>
			</array>
			<key>CFBundleURLName</key>
			<string>cl.micomercio.mi-app</string>
		</dict>
	</array>
```

Y en tu `AppDelegate` deberás manejar esas urls que comiencen con mi-app:

```swift
func application(_ app: UIApplication, open url: URL,
                 options: [UIApplicationOpenURLOptionsKey : Any] = [:]) -> Bool {

    let sendingAppID = options[.sourceApplication]
    print("source application = \(sendingAppID ?? "Unknown")")
    guard
        let components = NSURLComponents(url: url, resolvingAgainstBaseURL: true),
        let host = components.host,
        let path = components.path,
        let params = components.queryItems
        else {
            os_log("URL, host, path o params inválidos: %s", url.absoluteString)
            return false
    }
    if (host == "mi-app" && path == "onepay-result") {
        guard
            let occ = params.first(where: { $0.name == "occ" })?.value,
            let externalUniqueNumber = params.first(where: { $0.name == "externalUniqueNumber" })?.value
            else {
                os_log("Falta un parámetro occ o externalUniqueNumber: %s", url.absoluteString)
                return false
        }
        // Envia el occ y el externalUniqueNumber a tu backend para que
        // confirme la transacción
        print(occ, externalUniqueNumber)

    } else {
        // Otras URLs que quizás maneja tu app
    }
    return false

}
```

En Android debes registrar un Activity que responda a la URL registrada en dicho appScheme. Para eso debes configurar un intent filter a tu activity.

```xml
<activity ...>
  <intent-filter>
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <data android:scheme="mi-app" android:host="mi-app" android:path="onepay-result" />
  </intent-filter>
</activity>
```

Luego en ese Activity podrás obtener el resultado de la transacción a través del Intent:

```java
Intent intent = getIntent();
String occ = intent.getStringExtra("occ");
String externalUniqueNumber = intent.getStringExtra("externalUniqueNumber");
```
Finalmente deberás enviar el occ y el externalUniqueNumber a tu backend (quizás
quieras reusar el mismo `callbackUrl` que confirma transacciones en modo
checkout, pasando el status `"PRE_AUTHORIZED"` y algun parámetro adicional que
le haga saber a tu backend que devuelva JSON en lugar de dibujar una página web
con el comprobante).

Con eso has concluido la integración de Onepay, incluyendo sus cuatro componentes:
backend, frontend-web (soportando canales WEB y MOBILE) y las dos apps móviles (para el canal APP). ¡Felicitaciones!

## Credenciales del comercio

Para establecer las credenciales de tu comercio (y no seguir usando las
credenciales pre-configuradas que solo fucnionan en TEST), puedes realizarlo
así:

<div class="language-simple" data-multiple-language></div>

```java
import cl.transbank.onepay.Onepay;
//...

Onepay.setApiKey("api-key-entregado-por-transbank");
Onepay.setSharedSecret("secreto-entregado-por-transbank");
```

```php
use Transbank\Onepay\OnepayBase;

OnepayBase::setApiKey("api-key-entregado-por-transbank");
OnepayBase::setSharedSecret("secreto-entregado-por-transbank");
```

```csharp
using Transbank.Onepay;

Onepay.ApiKey = "api-key-entregado-por-transbank";
Onepay.SharedSecret = "secreto-entregado-por-transbank";
```

También puedes configurar el api key y secret de una petición específica:

<div class="language-simple" data-multiple-language></div>


```java
import cl.transbank.onepay.model.Options;
//...

Options options = new Options()
                  .setApiKey("api-key-entregado-por-transbank")
                  .setSharedSecret("secreto-entregado-por-transbank");
```


```php
use Transbank\Onepay\Options;

$options = new Options(
    'api-key-entregado-por-transbank',
    'secreto-entregado-por-transbank');
```

```csharp
var options = new Options()
        {
            ApiKey = "api-key-entregado-por-transbank",
            SharedSecret = "secreto-entregado-por-transbank"
        }
```

Esas opciones puedes pasarlas como parámetro a cualquier método
transaccional de Onepay (`Transaction.create`, `Transaction.commit` y
`Refund.create`).

### Apuntar a producción

Puedes configurar el SDK para utilizar los servicios del ambiente de `LIVE`
(Producción) de la siguiente forma:

<div class="language-simple" data-multiple-language></div>

```java
import cl.transbank.onepay.Onepay;
//...
Onepay.setIntegrationType(Onepay.IntegrationType.LIVE);
```

```php
use Transbank\Onepay\OnepayBase;

OnepayBase::setCurrentIntegrationType('LIVE');
```

```csharp
using Transbank.Onepay;

Onepay.IntegrationType = Transbank.Onepay.Enums.OnepayIntegrationType.LIVE;
```

## Más funcionalidades

Consulta la [referencia del API](/referencia/onepay) para más funcionalidades ofrecidas por Onepay:

- [Especificar tus propios external unique numbers](/referencia/onepay#especificar-tus-propios-external-unique-numbers) si quieres
usarlos para asociarlos a un número que ya exista en tu aplicación (pero
recuerda que si la transacción falla y quieres reintentar debes usar otro
external unique number).

- [Anular una transacción Onepay](/referencia/onepay#anular-una-transaccion) para revertir completamente una
transacción.

- [QR Directo](/referencia/onepay#modalidad-qr-directo) para controlar tú mismo
  el frontend web (javascript) durante todo el flujo. Necesitarás hacer mucho
  más trabajo, pero tendrás total control de la interacción con el usuario, con Onepay y con tu backend.

- [Detectar y/o instalar la app Onepay desde tu app](/referencia/onepay#detectar-e-instalar-la-app-onepay) para asegurar la
  mejor experiencia de pago en apps móviles.

<div class="container slate">
  <div class='slate-after-footer'>
    <div class='row d-flex align-items-stretch'>
      <div class='col-12 col-lg-6'>
        <h3 class='toc-ignore fo-size-22'>¿Tienes alguna duda de integración?</h3>
        <a href='https://join-transbankdevelopers-slack.herokuapp.com/' target='_blank'>
          <div class='td_block_gray'>
            <img src="https://p9.zdassets.com/hc/theme_assets/138842/200037786/logo.png" alt="" style="width: 90px; min-width: 100px;">
            <div class='td_pa-txt'>
              Únete a la comunidad de integradores en Slack y comparte buenos tips ayudando a los nuevos o buscando soluciones alternativas. Nuestro equipo está ahí para ayudarte.
            </div>
          </div>
        </a>
      </div>
      <div class='col-12 col-lg-6'>
        <h3 class='toc-ignore fo-size-22'>Si aún tienes dudas envíanos un mensaje</h3>
        <a class="pointer magenta" data-toggle='modal' data-target='#modalContactForm'>
          <div class='td_block_gray'>
            <div class="fo-size-20"><i class="fas fa-envelope"></i> Envianos un mensaje..</div>
            <div class='td_pa-txt'>
              Si necesitas resolver algún tipo de incidencia en el portal o si existe algún problema con tu integración y  que no has podido resolver, contáctanos a través de nuestro formulario.
            </div>
          </div>
        </a>
      </div>
    </div>
  </div>
</div>