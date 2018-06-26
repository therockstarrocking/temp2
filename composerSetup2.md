mkdir -p $HOME/composer/Lab

mkdir -p $HOME/composer/Provider

mkdir -p $HOME/composer/Lab


## Replace ca file 

awk 'NF {sub(/\r/, ""); printf "%s\\n",$0;}' crypto-config/peerOrganizations/Lab.example.com/peers/peer0.Lab.example.com/tls/ca.crt > $HOME/composer/Lab/ca-Lab.txt

awk 'NF {sub(/\r/, ""); printf "%s\\n",$0;}' crypto-config/peerOrganizations/Provider.example.com/peers/peer0.Provider.example.com/tls/ca.crt > $HOME/composer/Provider/ca-Provider.txt

awk 'NF {sub(/\r/, ""); printf "%s\\n",$0;}' crypto-config/ordererOrganizations/example.com/orderers/orderer.example.com/tls/ca.crt > $HOME/composer/ca-orderer.txt

## Getting Admin Keys 

export ORG1=crypto-config/peerOrganizations/Lab.example.com/users/Admin@Lab.example.com/msp

cp -p $ORG1/signcerts/A*.pem $HOME/composer/Lab

cp -p $ORG1/keystore/*_sk $HOME/composer/Lab

# for Providers

export ORG1=crypto-config/peerOrganizations/Provider.example.com/users/Admin@Provider.example.com/msp

cp -p $ORG1/signcerts/A*.pem $HOME/composer/Provider

cp -p $ORG1/keystore/*_sk $HOME/composer/Provider

## Creating business network cards for the Hyperledger Fabric administrator for Org1

composer card create -p $HOME/composer/Lab/byfn-network-Lab.json -u PeerAdmin -c $HOME/composer/Lab/Admin@Lab.example.com-cert.pem -k $HOME/composer/Lab/*_sk -r PeerAdmin -r ChannelAdmin -f PeerAdmin@byfn-network-Lab.card

## Creating business network cards for the Hyperledger Fabric administrator for Org2

composer card create -p $HOME/composer/Provider/byfn-network-Providerlc.json -u PeerAdmin -c $HOME/composer/Provider/Admin@Provider.example.com-cert.pem -k $HOME/composer/Provider/*_sk -r PeerAdmin -r ChannelAdmin -f PeerAdmin@byfn-network-Providerlc.card

## Importing the business network cards for the Hyperledger Fabric administrator for Org1

composer card import -f PeerAdmin@byfn-network-Lab.card --card PeerAdmin@byfn-network-Lab

##  Importing the business network cards for the Hyperledger Fabric administrator for Org2

composer card import -f PeerAdmin@byfn-network-Providerlc.card --card PeerAdmin@byfn-network-Providerlc

## Installing the Hyperledger Composer runtime onto the Hyperledger Fabric peer nodes for Org1

composer network install --card PeerAdmin@byfn-network-Lab --archiveFile trade-network.bna

## Installing the Hyperledger Composer runtime onto the Hyperledger Fabric peer nodes for Org2

composer network install --card PeerAdmin@byfn-network-Providerlc --archiveFile trade-network.bna
##  Defining the endorsement policy for the business network

##  Understanding and selecting the business network administrators

## Retrieving business network administrator certificates for Org1

composer identity request -c PeerAdmin@byfn-network-Lab -u admin -s adminpw -d john

## Retrieving business network administrator certificates for Org2

composer identity request -c PeerAdmin@byfn-network-Providerlc -u admin -s adminpw -d doe

## Starting the business network

composer network start -c PeerAdmin@byfn-network-Lab -n trade-network -V 0.2.4-deploy.0 -o endorsementPolicyFile=$HOME/composer/endorsement-policy.json -A john -C john/admin-pub.pem -A doe -C doe/admin-pub.pem
                     OR
composer network start -c PeerAdmin@byfn-network-Lab -n trade-network -V 0.2.4-deploy.0  -A john -C john/admin-pub.pem -A doe -C doe/admin-pub.pem


## Creating a business network card to access the business network as Org1

composer card create -p $HOME/composer/Lab/byfn-network-Lab.json -u john -n trade-network -c john/admin-pub.pem -k john/admin-priv.pem

composer card import -f john@trade-network.card
composer network ping -c john@trade-network

composer card create -p $HOME/composer/Provider/byfn-network-Providerlc.json -u doe -n trade-network -c doe/admin-pub.pem -k doe/admin-priv.pem

composer card import -f doe@trade-network.card
composer network ping -c doe@trade-network


composer participant add -c john@trade-network -d '{"$class":"org.acme.trading.Trader","tradeId":"trader1-org1", "firstName":"jim","lastName":"Doe"}'

composer identity issue -c john@trade-network -f jim.card -u jim -a "resource:org.acme.trading.Trader#trader1-org1"

composer card import -f jim.card

composer network ping -c jim@trade-network

composer transaction submit --card jim@trade-network -d '{"$class": "org.hyperledger.composer.system.AddAsset","registryType": "Asset","registryId": "org.acme.trading.Commodity", "targetRegistry" : "resource:org.hyperledger.composer.system.AssetRegistry#org.acme.trading.Commodity", "resources": [{"$class": "org.acme.trading.Commodity","tradingSymbol":"AG", "description":"Corn commodity","mainExchange":"EURONEXT", "quantity":"99","owner":"resource:org.acme.trading.Trader#trader1-org1"}]}'

composer network list -c jim@trade-network


composer participant add -c doe@trade-network -d '{"$class":"org.acme.trading.Trader","tradeId":"trader2-Provider", "firstName":"sam","lastName":"Lowe"}'

composer identity issue -c doe@trade-network -f sam.card -u sam -a "resource:org.acme.trading.Trader#trader2-Provider"

composer card import -f sam.card

composer network ping -c sam@trade-network

composer transaction submit --card john@trade-network -d '{"$class":"org.acme.trading.Trade","commodity":"resource:org.acme.trading.Commodity#EMA","newOwner":"resource:org.acme.trading.Trader#trader2-Provider"}'

composer network list -c sam@trade-network

## binding an identity 

composer card export -c alice@trade-network

composer identity bind --card sam@trade-network --participantId resource:org.hyperledger.composer.system.NetworkAdmin#admin --certificateFile ./alice/credentials/certificate           

composer card create -p $HOME/composer/Patient/byfn-network-Patient.json --businessNetworkName trade-network -u admin -c ./alice/credentials/certificate  -k ./alice/credentials/privateKey -f newsam1.card


composer card import --file newsam1.card --card newsam1

composer network ping --card newsam1

composer transaction submit --card alice@trade-network -d '{"$class": "org.acme.trading", "commodity": "resource:org.acme.trading.Commodity#EMA", "newOwner": "resource:org.acme.trading.Trader#trader1-org1"}'

composer transaction submit --card alice@trade-network -d '{"$class": "org.hyperledger.composer.system.AddAsset","registryType": "Asset","registryId": "org.acme.trading.Commodity", "targetRegistry" : "resource:org.hyperledger.composer.system.AssetRegistry#org.acme.trading.Commodity", "resources": [{"$class": "org.acme.trading.Commodity","tradingSymbol": "AGQ","owner": "resource:org.acme.trading.Trader#fred@example.com","description": "a lot of gold", "mainExchange": "exchange", "quantity" : 10}]}'

composer transaction submit --card alice@trade-network -d '{"$class":"org.acme.trading.Trade","commodity":"resource:org.acme.trading.Commodity#AGQ","newOwner":"resource:org.acme.trading.Trader#trader1-org1"}'

## binding providers participant 

composer card export -c dlowe@trade-network

composer identity bind --card doe@trade-network --participantId resource:org.acme.trading.Trader#trader1-org1 --certificateFile ./dlowe1/credentials/certificate        

composer card create -p $HOME/composer/Provider/byfn-network-Providerlc.json --businessNetworkName trade-network -u admin -c ./dlowe1/credentials/certificate  -k ./dlowe1/credentials/privateKey -f newdlowe.card   

composer card import --file newdlowe.card --card newdlowe

composer network ping --card newdlowe

composer transaction submit --card newdlowe -d '{"$class":"org.acme.trading.Trade","commodity":"resource:org.acme.trading.Commodity#AGQ","newOwner":"resource:org.acme.trading.Trader#trader1-org1"}'


