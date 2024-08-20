# CONFIGURACIÓN POSTMAN

En este directorio se entregan dos archivos, uno correspondiente a la plantilla o *template* para el entorno en Postman (<code>SF Environment Template.postman_environment.json</code>), donde se almacenarán los datos de configuración para la conexión con Salesforce, y otro con la colección de los diferentes llamados o solicitudes que se realizan para validar el correcto funcionamiento y configuración de las pruebas (<code>B2BOMS-8.postman_collection.json</code>).

## Pre-requisitos

Para poder conectarse correctamente por medio de las APIs de Salesforce, sean estas aquellas ya provistas por Salesforce, o creadas especificamente para un entorno determinado, es necesario contar con lo siguiente:

* Un usuario activo en el entorno de Salesforce, con permisos suficientes para poder hacer llamados hacia Salesforce.

* Una aplicación conectada configurada con los permisos necesarios, y que el usuario tenga a su asignado el uso de dicha aplicación conectada.

Para la integración se está utilizando inicialmente la aplicación conectada **AccountResAPI**.

## Plantilla de entorno

La plantilla de entorno se establecen las variables de entorno necesarias que las solicitudes funcionen correctamente, y de esta manera poder autorizar y conectar con entornos de Salesforce. Se recomienda duplicar el template entregado para la conexión con un nuevo entorno de Salesforce, dandole un nombre adecuada que identifique facilmente a qué entorno de Salesforce corresponde. Dichas variables son las siguientes:

* **Variables asociadas a la aplicación conectada (*Connected App*)**:
    * <code>sf_client_id</code>: *Hash* correspondiente al identificador de la aplicación conectada.
    * <code>sf_client_secret</code>: *Hash* correspondiente al secreto asociado a la aplicación conectada.

    Estos valores pueden ser los mismos para todos los usuarios de una misma organización, pero serán diferentes para cada una de las organizaciones de Salesforce, aún si se usa la misma aplicación conectada.

* **Variables asociadas al usuario de Salesforce**:
    * <code>sf_user_name</code>: Usuario del Salesforce.
    * <code>sf_password</code>: Contraseña del usuario de Salesforce.
    * <code>sf_user_token</code>: Token de seguridad asociado al usuario de Salesforce.

    Normalmente el usuario y su respectiva contraseña se entregan o configuran durante la creación del usuario. El token de seguridad será necesario generarlo la primera vez dentro de la configuración del usuario dentro de la plataforma de Salesforce.

* **Variables Asociadas al entorno de Salesforce**
    * <code>instance_url</code>: Corresponde a que tipo de organización se está tratando de acceder. Esta variable sera igual a <code>https://login.salesforce.com</code> si el entorno es uno productivo o *developer edition*, o será <code>https://test.salesforce.com</code> si el entorno es un *sandbox*.
    * <code>sf_url</code>: URL de la versión *classic* del entorno de Salesforce (https://\<<my-instance\>>.my.salesforce.com).
    * <code>access_token</code>: Token de autorización generado al realizar el llamado de autenticacíon y autorización (más adelante se ve este proceso).
    * <code>version</code>: Versión de API de Salesforce usada.

Si se necesita, siempre es posible agregar más variables de acuerdo a las necesidades y preferencias de cada usuario.

## Colección de llamados.

La colección de llamadas <code>B2BOMS-8.postman_collection.json</code> contiene diferentes carpetas agrupando los llamados necesarios para simular la integración con **SAP** que tendría Salesforce, y las acciónes que esta integración realiza sobre los diferentes registros de Salesforce.

Esta colección utiliza las siguientes variables de colección:

* <code>current_order_id</code>: Corresponde al Id en Salesforce de un registro del tipo `OrderSummary`.

* <code>random_id</code>: Cadena de caracteres usada para establecer un id falso que se asociará al campo `External_Reference_Id__c` del registro `OrderSummary`.

* <code>random_id2</code>: Cadena de caracteres usada para establecer un id falso que se asociará al campo `External_Reference_Id__c` del registro `FulfillmentOrder`.

* <code>today</code>: Texto que representa la fecha en formato DateTime (2024-08-20T00:00:00Z). Se utiliza para establecer el límite de métodos de delivery encontrados durante la ejecución de los Escenarios 2 y 5.

* <code>current_delivery_method_id</code>: Corresponde al Id en Salesforce de un registro del tipo `OrderDeliveryMethod`.

* <code>last_preparation_date</code>: Fecha establecida como la última de preparación de pedidos para los Escenarios 2 y 5. Actualmente está establecida para que sea dos días previos al día en curso.

* <code>current_fulfillment_order_id</code>: Corresponde al Id en Salesforce de un registro del tipo `FulfillmentOrder`.

Todas esta variables se actualizan automaticamente de acuerdo a los resultados de las solicitudes almacenadas en la colección.


### [POST] Get Auth Token

Este es el llamado que hay que realizar inicialmente para obtener el token de acceso a Salesforce. Si la configuración del entorno explicada previamente se hizo correctamente, y el usuario y la aplicación conectada tienen los permisos y configuraciones necesarias, sólo hará falta dar clic al botón *"Send"* y se obtendrá el token de acceso.

El llamado es de tipo *POST*, y su cuerpo está construido como se muestra a continuación:

| Key | Value |
|----------|:-------------:|
| grant_type | `"password"` |
| client_id |    `{{sf_client_id}}`   |
| client_secret | `{{sf_client_secret}}` |
| username | `{{sf_user_name}}` |
| password | `{{sf_password}}{{sf_user_token}}` |

donde los valores entre corchetes son determinados dinamicamente de acuerdo a lo establecido en el entorno, como ya se explicó previamente.

Si el llamado se ejecuta correctamente, la respuesta deberá ser similar a lo siguiente:

    {
        "access_token": "<<BIG HASH - YOUR TOKEN ID>>",
        "instance_url": "https://<<YOUR_INSTANCE>>.my.salesforce.com",
        "id": "https://login.salesforce.com/id/<<SOME_ID>>",
        "token_type": "Bearer",
        "issued_at": "<<1721588207274>>",
        "signature": "<<ANOTHER HASH>>="
    }

Como parte de la configuración del llamado, existe un `script` que se ejecuta después de llamado que es el siguiente:

    const jsonData = pm.response.json();
    const id = jsonData.id.split('/');

    if (!jsonData.error) {
        const context = pm.environment.name ? pm.environment : pm.collectionVariables;
        context.set("access_token", jsonData.access_token);
    }

Que basicamente establece la variable de entorno `access_token` con el valor de la variable `access_token` encontrada en el cuerpo de la respuesta.

### Escenarios 1 y 4

#### [GET] 1 - Get Order Summary

Este llamado se utiliza para buscar los registros `OrderSummary` asociados las solicitudes creadas en el Site por el número de solicitud correspondiente. La consulta es similar a esta:
```
SELECT Id,Submitter_Id__c,Has_Submitted_Cart_been_Updated__c FROM OrderSummary WHERE OriginalOrder.OrderNumber = '00001037'
```
Si existen un registro, la respuesta será algo como lo que sigue:
```
{
    "totalSize": 1,
    "done": true,
    "records": [
        {
            "attributes": {
                "type": "OrderSummary",
                "url": "/services/data/v60.0/sobjects/OrderSummary/1OsVZ0000000BDB0A2"
            },
            "Id": "1OsVZ0000000BDB0A2",
            "Submitter_Id__c": null,
            "Has_Submitted_Cart_been_Updated__c": false
        }
    ]
}
```
Esta llamdo establece el valor de la variable `current_order_id` mediante un *postscript*:

```javascript:
const jsonData = pm.response.json();
const id = jsonData.records[0].Id;

if (!jsonData.error) {
    const context = pm.collectionVariables;
    context.set("current_order_id", id);
}
```
#### [PATCH] 2 - Add External Reference Id

Este llamado toma el valor de la variable `current_order_id`, para actualizar el registro asociado, agregandole el campo ´External_Reference_Id__c´ con una cadena generada previamente mediante un *prescript*:
```javascript:
let result = '';
const characters = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789';
const charactersLength = characters.length;
let counter = 0;
while (counter < 8) {
    result += characters.charAt(Math.floor(Math.random() * charactersLength));
    counter += 1;
}
pm.collectionVariables.set("random_id", result)
```
El cuerpo de la solicitud se similar al siguiente:
```
{
    "External_Reference_Id__c": "{{random_id}}"
}
```
No genera ningún cuerpo de respuesta, pero el código de la solicitud deberá ser `204` para saber que la actualización se ejecutó correctamente.

### Escenarios 2 y 5

#### [GET] 1 - Get Order Summary

Exactamente igual que **[GET] 1 - Get Order Summary** para los escenarios 1 y 4.

#### [GET] 2 - Get Delivery Method Id

Este llamado se encarga de recuperar de Salesforce el tipo de delivery que se haya creado durante el día de hoy. La consulta realizada es similar a la siguiente:
```
SELECT Id, Name,CreatedDate FROM OrderDeliveryMethod WHERE CreatedDate >= {{today}}
```
donde `{{today}}` es el valor de la variable de colección. Se puede modificar dicha variable para ser muy específico con el tiempo despúes del cual filtrar los registros. Se recomienda igualmente tratar de no tener registros previos al momento realizar las pruebas.

SI el llamdo funciona bien, deberá retornar una lista (idealmente de un solo elemento) comola siguiente:
```
{
    "totalSize": 2,
    "done": true,
    "records": [
        {
            "attributes": {
                "type": "OrderDeliveryMethod",
                "url": "/services/data/v60.0/sobjects/OrderDeliveryMethod/2DmVZ0000004cLF0AY"
            },
            "Id": "2DmVZ0000004cLF0AY",
            "Name": "Retiro en CD",
            "CreatedDate": "2024-08-20T13:24:55.000+0000"
        }
    ]
}
```
#### [PATCH] 3 - Add External Reference Id

Exactamente igual a **[PATCH] 2 - Add External Reference Id** para los escenarios 1 y 4.

#### [POST] 4 - Create Fulfilled Order

Este llamado clera un registro del tipo `FulfillmentOrder` en Salesforce, asociandolo a registro `OrderSummary` consultado anteriormente mediante el llamado **[GET] 1 - Get Order Summary** y al registro `OrderDeliveryMethod` consultado previamente usando el llamado **[GET] 2 - Get Delivery Method Id**. El cuerpo de la solicitud será similar a:
```
{
    "OrderSummaryId": "{{current_order_id}}",
    "Status": "Borrador",
    "FulfilledToName": "Test",
    "VehicleLicense__c": "AACL99",
    "Transport_Number__c": 1112233,
    "External_Reference_Id__c": "{{random_id2}}",
    "DeliveryMethodId" : "{{current_delivery_method_id}}",
    "LastPreparationDate__c": "{{last_preparation_date}}"
}
```
La fecha para la variable `last_preparation_date`, y el valor de la variable `random_id2` el se establecen en un *prescript* asociado a este llamado:
```javascript:
let result = '';
const characters = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789';
const charactersLength = characters.length;
let counter = 0;
while (counter < 6) {
    result += characters.charAt(Math.floor(Math.random() * charactersLength));
    counter += 1;
}
pm.collectionVariables.set("random_id2", '50' + result)
let lastPreparationDate = new Date(new Date().setDate(new Date().getDate() - 2));
pm.collectionVariables.set('last_preparation_date', lastPreparationDate.toISOString())
```
Si la creación funciona correctamente, se obtendrá una respuesta parecida a la siguiente:
```
{
    "id": "a2FVZ00000056ZR2AY",
    "success": true,
    "errors": []
}
```

### Escenario 3

#### [GET] 1 - Get Order Summary

Exactamente igual que **[GET] 1 - Get Order Summary** para los escenarios 1 y 4.

#### [PATCH] 2 - Add External Reference Id

Exactamente igual a **[PATCH] 2 - Add External Reference Id** para los escenarios 1 y 4.

#### [POST] 3 - Create Fulfilled Order

Similar al llamado **[POST] 4 - Create Fulfilled Order** para los escenarios 2 y 5 pero con un cuerpo diferente, a saber:
```
{
    "OrderSummaryId": "{{current_order_id}}",
    "Status": "Borrador",
    "FulfilledToName": "Test"
}
```
#### [PATCH] 4 - Update Fulfillment Order

Este llamado actualiza el registro creado durante el llamado anterior **[POST] 3 - Create Fulfilled Order**. El cuerpo para la actualización es el siguiente:

```
{
    "VehicleLicense__c": "AACL99",
    "Transport_Number__c": 1112233,
    "External_Reference_Id__c": "{{random_id2}}"
}
```
Si la actualización es correcta, no debería llegar ningún cuerpo de respuesta, pero el código de la solicitud debe ser `204`.