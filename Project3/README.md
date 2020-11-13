

# Proyecto 3: Instalar Hyperledger Explorer

Para más información sobre este curso visite el siguiente enlace: https://learn.coding-bootcamps.com/p/blockchain-hyperledger-fabric-para-el-administrador-del-sistema 

En base al proyecto 1 instalaremos Hyperledger Explorer, donde podremos ver de una manera visual el funcionamiento de nuestra red.

1. Iniciar y preparar la red test-network de Hyperledger Fabric.
2. Instalar y configurar Hyperledger Explorer
3. Iniciar Hyperledger Explorer
4. Parar Hyperledger Explorer


## Prerequisitos

- Realizar el proyecto 1.
- Clonar el repositorio de Hyperledger Explorer: https://github.com/hyperledger/blockchain-explorer 


## 1. Iniciar y preparar la red test-network de Hyperledger Fabric

Accedemos al directorio utilizado en el proyecto 1 en la carpeta `test-network`.
Iniciamos la red con un canal utilizando el comando:
`./network.sh up createChannel`

## 2. Instalar y configurar Hyperledger Explorer

Para instalar e iniciar Hyperledger Explorer utilizaremos Docker y Docker Compose. Hyperledger nos proporciona las imágenes necesarias para poder hacerlo.

Accedemos al directorio de nuestra red `test-network` de HF y accedemos a la carpeta `organizations`:
`cd fabric-samples/test-network/organizations`

Copiamos las carpetas `ordererOrganizations` y `peerOrganizations` y las pegamos en el directorio de nuestra carpeta `crypto` de Hyperledger Explorer:
`blockchain-explorer/examples/net1/crypto`

Básicamente lo que estamos haciendo es copiar el material criptográfico de nuestras organizaciones en la carpeta de Hyperledger Explorer para que lo pueda utilizar.

En la carpeta `net1` tendriamos que tener la siguiente estructura de carpetas:
`config.json`
`connection-profile/first-network.json`
`crypto` Contiene el material criptográfico que hemos copiado de la red de Hyperledger Fabric.

Accedemos al repositorio clonado de Hyperledger Explorer: `cd blockchain-explorer `
En el archivo `docker-compose.yaml`, revisamos que tenemos la siguiente configuración establecida:
`
 networks:
    mynetwork.com:
        external:
            name: net_test

    ...

    services:
      explorer.mynetwork.com:

        ...

        volumes:
          - ./config.json:/opt/explorer/app/platform/fabric/config.json
          - ./connection-profile:/opt/explorer/app/platform/fabric/connection-profile
          - ./organizations:/tmp/crypto
          - walletstore:/opt/wallet
`

En el archivo `connection-profile/first-network.json` revisamos que los directorios relacionados con los certificados, claves, etc de las organizaciones y peers sean correctos.


## 3. Iniciar Hyperledger Explorer

En el directorio `/blockchain-explorer`, ejecutamos el siguiente comando para iniciar Hyperledger Explorer:
`docker-compose up -d`

Podemos acceder a la interfaz visual de Hyperledger Explorer en: 
`localhost:8080`
Para poder entrar nos solicitaran usuario y contraseña, esta información la podemos encontrar en el archivo `first-network.json` en el apartado `client.adminCredential`.

Podremos ver de forma visual información de nuestra red, bloques, transaciones, nodos, chaincodes, etc. 

Si interactuamos con nuestra red `test-network` (como hemos hecho en el proyecto 1), podremos ver en Hyperledger Explorer como se crean los bloques, transacciones, etc.


## 4. Parar Hyperledger Explorer

Para parar el servicio de Hyperledger Explorer utilizamos el comando: `docker-composer down`

