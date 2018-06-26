mkdir -p $HOME/composer/Patient

mkdir -p $HOME/composer/Provider


## Replace ca file 

awk 'NF {sub(/\r/, ""); printf "%s\\n",$0;}' crypto-config/peerOrganizations/Patient.example.com/peers/peer0.Patient.example.com/tls/ca.crt > $HOME/composer/Patient/ca-patient.txt

awk 'NF {sub(/\r/, ""); printf "%s\\n",$0;}' crypto-config/peerOrganizations/Provider.example.com/peers/peer0.Provider.example.com/tls/ca.crt > $HOME/composer/Provider/ca-Provider.txt

awk 'NF {sub(/\r/, ""); printf "%s\\n",$0;}' crypto-config/ordererOrganizations/example.com/orderers/orderer.example.com/tls/ca.crt > $HOME/composer/ca-orderer.txt

## Getting Admin Keys 

export ORG1=crypto-config/peerOrganizations/Patient.example.com/users/Admin@Patient.example.com/msp

cp -p $ORG1/signcerts/A*.pem $HOME/composer/Patient

cp -p $ORG1/keystore/*_sk $HOME/composer/Patient

# for Providers

export ORG1=crypto-config/peerOrganizations/Provider.example.com/users/Admin@Provider.example.com/msp

cp -p $ORG1/signcerts/A*.pem $HOME/composer/Provider

cp -p $ORG1/keystore/*_sk $HOME/composer/Provider

## Creating business network cards for the Hyperledger Fabric administrator for Org1

composer card create -p $HOME/composer/Patient/byfn-network-Patient.json -u PeerAdmin -c $HOME/composer/Patient/Admin@Patient.example.com-cert.pem -k $HOME/composer/Patient/*_sk -r PeerAdmin -r ChannelAdmin -f PeerAdmin@byfn-network-Patient.card

## Creating business network cards for the Hyperledger Fabric administrator for Org2

composer card create -p $HOME/composer/Provider/byfn-network-Provider.json -u PeerAdmin -c $HOME/composer/Provider/Admin@Provider.example.com-cert.pem -k $HOME/composer/Provider/*_sk -r PeerAdmin -r ChannelAdmin -f PeerAdmin@byfn-network-Provider.card

## Importing the business network cards for the Hyperledger Fabric administrator for Org1

composer card import -f PeerAdmin@byfn-network-Patient.card --card PeerAdmin@byfn-network-Patient

##  Importing the business network cards for the Hyperledger Fabric administrator for Org2

composer card import -f PeerAdmin@byfn-network-Provider.card --card PeerAdmin@byfn-network-Provider

## Installing the Hyperledger Composer runtime onto the Hyperledger Fabric peer nodes for Org1

composer network install --card PeerAdmin@byfn-network-Patient --archiveFile trade-network.bna

## Installing the Hyperledger Composer runtime onto the Hyperledger Fabric peer nodes for Org2

composer network install --card PeerAdmin@byfn-network-Provider --archiveFile trade-network.bna
##  Defining the endorsement policy for the business network

##  Understanding and selecting the business network administrators

## Retrieving business network administrator certificates for Org1

composer identity request -c PeerAdmin@byfn-network-Patient -u admin -s adminpw -d alice

## Retrieving business network administrator certificates for Org2

composer identity request -c PeerAdmin@byfn-network-Provider -u admin -s adminpw -d bob

## Starting the business network

composer network start -c PeerAdmin@byfn-network-Patient -n trade-network -V 0.2.4-deploy.0 -A alice -C alice/admin-pub.pem -A bob -C bob/admin-pub.pem

## Creating a business network card to access the business network as Org1

composer card create -p $HOME/composer/Patient/byfn-network-Patient.json -u alice -n trade-network -c alice/admin-pub.pem -k alice/admin-priv.pem

composer card import -f alice@trade-network.card
composer network ping -c alice@trade-network

## Creating a business network card to access the business network as Org2

composer card create -p $HOME/composer/Provider/byfn-network-Provider.json -u bob -n trade-network -c bob/admin-pub.pem -k bob/admin-priv.pem

composer card import -f bob@trade-network.card
composer network ping -c bob@trade-network

composer participant add -c bob@trade-network -d '{"$class":"org.acme.trading.Trader","tradeId":"trader2-Provider", "firstName":"Dave","lastName":"Lowe"}'

composer identity issue -c bob@trade-network -f dave.card -u dlowe -a "resource:org.acme.trading.Trader#trader2-Provider"

composer card import -f dave.card

composer network ping -c dlowe@trade-network

composer transaction submit --card jdoe@trade-network -d '{"$class":"org.acme.trading.Trade","commodity":"resource:org.acme.trading.Commodity#EMA","newOwner":"resource:org.acme.trading.Trader#trader2-Provider"}'

composer network list -c dlowe@trade-network

composer transaction submit --card alice@trade-network -d '{"$class": "org.hyperledger.composer.system.AddAsset","registryType": "Asset","registryId": "org.acme.trading.Commodity", "targetRegistry" : "resource:org.hyperledger.composer.system.AssetRegistry#org.acme.trading.Commodity", "resources": [{"$class": "org.acme.trading.Commodity","tradingSymbol": "AGQ","owner": "resource:org.acme.trading.Trader#trader-org1","description": "a lot of gold", "mainExchange": "exchange", "quantity" : 10}]}'
