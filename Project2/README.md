

# Proyecto 2: Añadir una Organización a una red de Hyperledger Fabric

En base al proyecto 1 añadiremos una nueva organización org3 a la red.

1. Iniciar Org3 en el canal
2. Generar el material criptográfico de la Org3
3. Iniciar los componentes de Org3
4. Preparar el entorno CLI
5. Preparar la configuración
6. Convertir la configuración a JSON
7. Añadir el material criptográfico de la Org3
8. Firmar y enviar la configuración actualizada
9. Unir Org3 al canal

## Prerequisitos

En base al proyecto 1, ejectuamos los siguientes comando: 
`./network.sh down` para limpiar el entorno.

`./network.sh up createChannel` para arrancar la red con un canal llamado mychannel.


## 1. Iniciar Org3 en el canal

En la carpeta `test-network` accedemos al directorio addOrg3 y ejecutamos el comando: `./addOrg3.sh up`. Se nos generará el material criptográfico, se actualizará la configuración del canal y se enviará al canal. Ya tendriamos la Org3 unida al canal. 

Este proceso se podría hacer de forma manual. A continuación ejectuamos los siguientes comandos para volver a preparar la red de nuevo: 
`./network.sh down`
`./network.sh up createChannel`

A continuación se muestran los pasos necesarios para añadir la Org3 a la red.

## 2. Generar el material criptográfico de la Org3

Abrimos una nueva terminal y accedemos al directorio addOrg3
`cd addOrg3`

Creamos loa certificados y claves para el peer de la Org3. Utilizaremos la herramienta cryptogen que leerá el archivo org3-crypto.yaml y generará el material criptográfico en la carpeta org3.example.com.
`../../bin/cryptogen generate --config=org3-crypto.yaml --output="../organizations"`

Podemos encontrar el material criptográfico de las organizaciones en el directorio: 
`test-network/organizations/peerOrganizations`

Utilizamos el siguiente comando y la herramienta configtxgen para imprimir la definición de la Org3.
`
export  FABRIC_CFG_PATH=$PWD  
../../bin/configtxgen -printOrg Org3MSP > ../organizations/peerOrganizations/org3.example.com/org3.json
`

## 3. Iniciar los componentes de Org3
Una vez creado el material criptográfico ya podemos arrancar el Org3 peer. 
Accedemos al directorio addOrg3 y ejecutamos el comando: 
`docker-compose -f docker/docker-compose-org3.yaml up -d`



## 4. Preparar el entorno CLI
Utilizamos el siguiente comando para ejecutar el contenedor Org3CLI:
`docker exec -it Org3cli bash`

El contenedor nos proporciona acceso al material criptográfico y los certificados TLS para todas las organizaciones. Podemos usar las variables de entorno para operar el contendor Org3CLI como administador de Org1, Org2 o Org3.

`export  ORDERER_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem`
`export  CHANNEL_NAME=mychannel`


Podemos comprar que las variables de han configurado correctamente con el comando: 
`echo  $ORDERER_CA && echo  $CHANNEL_NAME`

## 5. Preparar la configuración

Para preparar la configuración del canal, necesitamos operar como administrador. Como la Org3 no está aún unida al canal, utilizaremos la Org1 para realizar la configuración.
`export  CORE_PEER_LOCALMSPID="Org1MSP"`
`export  CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt`
`export  CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/organizations/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp`
`export  CORE_PEER_ADDRESS=peer0.org1.example.com:7051`


Utilizamos el siguiente comando para buscar el último bloque de configuración:
`peer channel fetch config config_block.pb -o orderer.example.com:7050 -c $CHANNEL_NAME --tls --cafile $ORDERER_CA`


## 6. Convertir la configuración a JSON

Utilizaremos la herramienta `configtxlator` para descodificar la configuración bloque del canal en formato JSON.
` configtxlator proto_decode --input config_block.pb --type  common.Block | jq .data.data[0].payload.data.config > config.json`

El comando nos proporciona el archivo config.json.

## 7. Añadir el material criptográfico de la Org3

Utilizamos la herramienta `jq` para añadir la configuración de la Org3 al canal.
`jq -s '.[0] * {"channel_group":{"groups":{"Application":{"groups": {"Org3MSP":.[1]}}}}}' config.json ./organizations/peerOrganizations/org3.example.com/org3.json > modified_config.json`

Tendremos el archivo config.json con los materiales de la Org1 y la Org2 y el archivo modified_config.json contiene las 3 organizaciones.

Traducimos los archivos al formato protobuf.
`configtxlator proto_encode --input config.json --type common.Config --output config.pb`
`configtxlator proto_encode --input modified_config.json --type common.Config --output modified_config.pb`

Utilizando configtxlator podemos analizar la diferencia entre los dos archivos de configuración protobufs.
`configtxlator compute_update --channel_id $CHANNEL_NAME --original config.pb --updated modified_config.pb --output org3_update.pb`

El nuevo archivo org3_update.pb, tiene la información de Org3, Org2 y Org1.
Antes de actualizar el canal, convertimos el archivo a un formato JSON. 
`configtxlator  proto_decode  --input  org3_update.pb  --type  common.ConfigUpdate  |  jq  . > org3_update.json`

A continuación envolvemos en archivo en un mensaje de "sobre":
`echo '{"payload":{"header":{"channel_header":{"channel_id":"'$CHANNEL_NAME'", "type":2}},"data":{"config_update":'$(cat org3_update.json)'}}}' | jq . > org3_update_in_envelope.json`

Con el archivo JSON preparado, utilizamos la herramienta configtxlator para convertir el archivo al formato protobuf, que es el formato que Fabric necesita.
`configtxlator proto_encode --input org3_update_in_envelope.json --type common.Envelope --output org3_update_in_envelope.pb`


## 8. Firmar y enviar la configuración actualizada

Necesitamos las firmas para poder enviar la nueva configuración. Nuestras politicas indican que necesitiamos una majoria para poder firmar los cambios. Necesitamos la Org1 y Org2 para poder realizar los cambios, sin las dos firmas el ordering service rechazará la transacción por no cumplir con las políticas.

Primero firmaremos la actualización como Org1. Importante exportar las variables de entorno necesarias para operar la Org3CLI como la Org1. Seguidamente ejecutamos el comando:
`peer channel signconfigtx -f org3_update_in_envelope.pb`

Seguidamente exportamos las variables de entorno para operar como Org2:
`export  CORE_PEER_LOCALMSPID="Org2MSP"`
`export  CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt`
`export  CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/organizations/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp`
`export  CORE_PEER_ADDRESS=peer0.org2.example.com:9051`

Con el siguiente comando actualizamos la configuración del canal:
`peer channel update -f org3_update_in_envelope.pb -c $CHANNEL_NAME -o orderer.example.com:7050 --tls --cafile $ORDERER_CA`

La nueva actualización se realizará en el bloque 3. En una nueva terminal podemos comprobar los logs:
`docker logs -f peer0.org1.example.com`

## 9. Unir Org3 al canal

En este punto la configuración del canal ha sido actualizara para poder incluir la nueva organización.
Exportamos la variables de entorno para operar como la Org3. Con el siguiente comando llamamos al ordering sercive y solicitamos el bloque genesis del canal. El ordering service verificará que la Org3 si puede unirse al canal.
` peer channel fetch 0 mychannel.block -o orderer.example.com:7050 -c $CHANNEL_NAME --tls --cafile $ORDERER_CA`

Finalmente usamos el siguiente canal para unir el peer de la Org3 al canal.
`peer channel join -b mychannel.block`

