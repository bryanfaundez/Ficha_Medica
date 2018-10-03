# Ficha medica Blockchain

Hola , este es mi blockchain para realizar fichas medicas con la seguridad de blockchain , esta realizada mediante hyperledger composer


# Pasos previos 

Antes de comenzar todo, prepare su entorno de desarrollo , siga los pasos de la pagina oficial de hyperledger [Instalacion requisitos previos](https://hyperledger.github.io/composer/latest/installing/installing-prereqs.html)

# Comandos de configuración del entorno

### Descargue el cliente y las herramientas de Hyperledger (o `npm update`si ya lo ha instalado)

 ```
npm install -g composer-cli
npm install -g generator-hyperledger-composer
npm install -g composer-rest-server
npm install -g yo
```

### Limpie los contenedores de docker (¡cuidado si utiliza los contenedores de docker para otros proyectos!)

   ```
docker kill $(docker ps -q)
docker rm $(docker ps -aq)
docker rmi $(docker images dev-* -q)
```

### Limpie las credenciales del composer y los perfiles de conexión que haya creado anteriormente (de nuevo, ¡esto supone que no tiene ningún proyecto importante de compositor ya iniciado!)

```
rm -rf ~/.composer-connection-profiles
rm -rf ~/.composer-credentials
```

### Inicie Hyperledger Fabric (esto es a lo que se conecta la red de negocios y es una serie de contenedores Docker).

```
mkdir fabric-tools
cd fabric-tools
curl -O https://raw.githubusercontent.com/hyperledger/composer-tools/master/packages/fabric-dev-servers/fabric-dev-servers.zip
unzip fabric-dev-servers.zip
./downloadFabric.sh
./startFabric.sh
./createPeerAdminCard.sh
```

##  Paso uno: Crear una estructura de red de negocios

![Diagrama de Hyperledger Composer](https://hyperledger.github.io/composer/latest/assets/img/Composer-Diagram.svg)

El concepto clave para Hyperledger Composer es la **definición de red de negocios (BND)** . Define el modelo de datos, la lógica de transacción y las reglas de control de acceso para su solución de blockchain. Para crear un BND, necesitamos crear una estructura de proyecto adecuada en el disco.

La forma más fácil de comenzar es usar el generador de Yeoman para crear una red de negocios de esqueleto. Esto creará un directorio que contiene todos los componentes de una red de negocios.

1.  Crear una red de negocios de esqueleto utilizando Yeoman. Este comando requerirá un nombre de red comercial, descripción, nombre de autor, dirección de correo electrónico del autor, selección de licencia y espacio de nombres.
```
yo hyperledger-composer:businessnetwork
```
1.  Entrar `tutorial-network`  para el nombre de la red y la información deseada para la descripción, el nombre del autor y el correo electrónico del autor.
    
2.  Seleccionar `Apache-2.0`  como la licencia.
    
3.  Seleccionar `org.ficha.life`  como el espacio de nombres.
    
4.  Seleccionar `No`  cuando se le pregunta si generar una red vacía o no.

### Paso Dos: Definiendo una red de negocios
Una red de negocios está formada por activos, participantes, transacciones, reglas de control de acceso y, opcionalmente, eventos y consultas. En la red empresarial esquelética creada en los pasos anteriores, hay un `.cto`archivo model ( ) que contendrá las definiciones de clase para todos los activos, participantes y transacciones en la red empresarial. La red empresarial principal también contiene un `permissions.acl`documento de control de acceso ( ) con reglas básicas de control de acceso, un `logic.js`archivo script ( ) que contiene funciones de procesador de transacciones y un `package.json`archivo que contiene metadatos de la red empresarial.

#### Modelado de activos, participantes y transacciones.

El primer documento a actualizar es el `.cto` archivo modelo ( ). Este archivo se escribe utilizando el [lenguaje de modelado de Hyperledger Composer](https://hyperledger.github.io/composer/latest/reference/cto_language.html) . El archivo modelo contiene las definiciones de cada clase de activo, transacción, participante y evento. Extiende implícitamente el modelo de sistema Hyperledger Composer descrito en la documentación del lenguaje de modelado.

1.  Abre el archivo modelo `org.ficha.life.cto`  
   2.  Reemplace el contenido con lo siguiente:
```
namespace  org.ficha.life
participant  Paciente  identified by  rut {
	  //o String codigoPaciente 
		o  String  rut
		o  String  nombre
		o  String  apellido
		o  Integer  edad
		o  String  afiliacion
		o  String  lugarResidencia
		o  Integer  numeroContacto
		o  String  ocupacion
}
participant  Medico  identified by  rut {
		//o String codigoMedico
		o  String  rut
		o  String  nombre
		o  String  apellido
		o  Integer  edad
		o  String  centroMedico
		o  String  especialidad
}
// activo , es como la data que ingresa al paciente por medio de la clave primaria codigo
asset  Ficha  identified by  codficha {
		o  String  codficha
		o  String  diagnostico
		o  String  cirujia
		o  String  medicamento
		o  String  factoresRiesgo
		o  String  enfermedadCronica
		-->Paciente  paciente
		-->Medico  medico
}
@commit(false)
@returns(Ficha[])
transaction  MyTransaction {
}
```
3. Guarda tus cambios en `org.ficha.life.cto`.
#### Añadiendo lógica de transacción de JavaScript
En el archivo modelo, `myTransaction`  muestra todos los asset 

1.  Abre el `logic.js`  archivo de comandos.
    
2.  Reemplace el contenido con lo siguiente:
```
'use strict';
async  function  myTransaction(transaction) {
	const  allAssets  = [];
	const  assetRegistry  =  await  getAssetRegistry('org.ficha.life.Ficha');
	const  localAssets  =  await  assetRegistry.getAll();
	for (const  asset  of  localAssets) {
		localAssets.push(asset);
	}
	const  businessNetworkNames  = ['other-network-1', 'other-network-2'];
	for (const  businessNetworkName  of  businessNetworkNames) {
		const  response  =  await  getNativeAPI().invokeChaincode(businessNetworkName, ['getAllResourcesInRegistry', 'Asset', 'org.ficha.life.Ficha'], 'composerchannel');
		const  json  =  JSON.parse(response.payload.toString('utf8'));
	for (const  item  of  json) {
		allAssets.push(getSerializer().fromJSON(item));
	}
}
return  allAssets;
}
```
3. Guarda tus cambios en `logic.js`.

#### Añadiendo control de acceso

1. Reemplace las siguientes reglas de control de acceso en el archivo `permissions.acl`  :
```
rule OwnRecordFullAccess {
	description: "Allow all participants full access to their own record"
	participant(p): "org.ficha.life.Paciente"
	operation: ALL
	resource(r): "org.ficha.life.Paciente"
	condition: (r.getIdentifier() ===  p.getIdentifier())
	action: ALLOW
}
rule OwnRecordFullAccess1 {
	description: "Allow all participants full access to their own record"
	participant(p): "org.ficha.life.Medico"
	operation: ALL
	resource(r): "org.ficha.life.Medico"
	condition: (r.getIdentifier() ===  p.getIdentifier())
	action: ALLOW
}
rule historianAccess{
	description: "Only allow members to read historian records referencing transactions they submitted."
	participant(p): "org.ficha.life.Paciente"
	operation: READ
	resource(r): "org.ficha.life.Ficha"
	condition: (r.paciente.getIdentifier() ==  p.getIdentifier())
	action: ALLOW
}
rule SystemACL {
	description: "System ACL to permit all access"
	participant: "org.hyperledger.composer.system.Participant"
	operation: ALL
	resource: "org.hyperledger.composer.system.**"
	action: ALLOW
}
rule NetworkAdminUser {
	description: "Grant business network administrators full access to user resources"
	participant: "org.hyperledger.composer.system.NetworkAdmin"
	operation: ALL
	resource: "**"
	action: ALLOW
}
rule NetworkAdminSystem {
	description: "Grant business network administrators full access to system resources"
	participant: "org.hyperledger.composer.system.NetworkAdmin"
	operation: ALL
	resource: "org.hyperledger.composer.system.**"
	action: ALLOW
}
```
3. Guarda tus cambios en `permissions.acl`.
## Paso tres: generar un archivo de red de negocios

Ahora que se ha definido la red de negocios, se debe empaquetar en un archivo de archivo de red de negocios desplegable ( `.bna`).

1.  Usando la línea de comando, navega a la `tutorial-network`  directorio.
2.  Desde el `tutorial-network`directorio, ejecute el siguiente comando:
    ```
    composer archive create -t dir -n . 
    ```
    Después de que se haya ejecutado el comando, `tutorial-network@0.0.1.bna`se ha creado un archivo de red de negocios llamado en el `tutorial-network`directorio.

## Paso cuatro: Despliegue de la red de negocios
Después de crear el `.bna`archivo, la red empresarial se puede implementar en la instancia de Hyperledger Fabric. Normalmente, la información del administrador de Fabric se requiere para crear una `PeerAdmin`identidad, con privilegios para instalar Chaincode en el par, así como para iniciar Chaincode en el `composerchannel`canal. Sin embargo, como parte de la instalación del entorno de desarrollo, `PeerAdmin`ya se ha creado una identidad.

Después de que se haya instalado la red de negocios, se puede iniciar la red. Para las mejores prácticas, se debe crear una nueva identidad para administrar la red de negocios después de la implementación. Esta identidad se conoce como un administrador de red.

#### Recuperando las credenciales correctas

Una `PeerAdmin`tarjeta de red de negocios con las credenciales correctas ya está creado como parte de la instalación del entorno de desarrollo.

#### Despliegue de la red empresarial

La implementación de una red de negocios en el Hyperledger Fabric requiere que la red de negocios de Hyperledger Composer se instale en el interlocutor, luego se puede iniciar la red de negocios y se debe crear una nueva tarjeta de participante, identidad y asociada para que sea el administrador de la red. Finalmente, la tarjeta de red empresarial del administrador de la red debe importarse para su uso, y luego se puede hacer ping a la red para verificar que está respondiendo.

1.  Para instalar la red de negocios, desde el directorio `tutorial-network`, ejecute el siguiente comando:
    ```
    composer network install --card PeerAdmin@hlfv1 --archiveFile tutorial-network@0.0.1.bna    
    ```
     El comando `composer network install`requiere una tarjeta de red comercial PeerAdmin (en este caso, una se ha creado e importado de antemano) y la ruta del archivo `.bna`que define la red empresarial.
    
2.  Para iniciar la red de negocios, ejecute el siguiente comando:
    ```
    composer network start --networkName tutorial-network --networkVersion 0.0.1 --networkAdmin admin --networkAdminEnrollSecret adminpw --card PeerAdmin@hlfv1 --file networkadmin.card
    ```
      El comando  `composer network start` requiere una tarjeta de red de negocios, así como el nombre de la identidad de administrador de la red de negocios, el nombre y la versión de la red de negocios y el nombre del archivo que se creará listo para importar como una tarjeta de red de negocios.
    
3.  Para importar la identidad del administrador de red como una tarjeta de red comercial utilizable, ejecute el siguiente comando: 
    ```
    composer card import --file networkadmin.card
    ```
    El comando `composer card import` requiere el nombre de archivo especificado `composer network start` para crear una tarjeta.
    
4.  Para verificar que la red empresarial se haya implementado correctamente, ejecute el siguiente comando para hacer ping a la red:
    ```
    composer network ping --card admin@tutorial-network  
    ```
El  comando `composer network ping`  requiere una tarjeta de red comercial para identificar la red para hacer ping.

## Paso Cinco: Generando un servidor REST

Hyperledger Composer puede generar una API REST a medida basada en una red empresarial. Para desarrollar una aplicación web, la API REST proporciona una capa útil de abstracción independiente del lenguaje.

1.  Para crear la API REST, navegue a la `tutorial-network`  directorio y ejecute el siguiente comando:
    ```
     composer-rest-server
    ```
    
2.  Entrar `admin@tutorial-network`  como el nombre de la tarjeta.
    
3.  Seleccionar **never  use namespaces** cuando se le pregunte si desea usar espacios de nombres en la API generada.
    
4.  Seleccionar **No** cuando se le pregunta si proteger la API generada.
    
5.  Seleccionar **Sí** cuando se le pregunta si desea habilitar la publicación de eventos.
    
6.  Seleccionar **No** cuando se le pregunta si desea habilitar la seguridad TLS.
   
La API generada está conectada a la cadena de bloques desplegada y a la red empresarial.
 **Sí no le aparece igual guiarse con la siguiente imagen**
 
![](https://lh3.googleusercontent.com/6iKf23EyBKS0jOtFNuMZrclxKuCc_z_JqiVuU694JaR0f482TcuDpHZDTq4e1sPBQYOXMy9FTdTH)
 

<!--stackedit_data:
eyJoaXN0b3J5IjpbMTU3NDY4MDU3OF19
-->