<h1 align="center"> Data Provider Setup </h1>

[TOC]

# Prerequisites

## Decentralized Identifier

Every response from the Data Provider (DP) needs to be signed with the associated decentralized identifier (DID), so Orange can verify it in case of any MITM attack. 

Process of setting up the DID (in Go): 

```go
remoteURL := "http://polaris2.ont.io:20336" //testnet RPC, for mainnet it should be "http://dappnode2.ont.io:20336"
sdk := ontsdk.NewOntologySdk()
sdk.NewRpcClient().SetAddress(remoteURL)
wlt, err := sdk.OpenWallet("/path/to/wallet.dat") // ./ontology account add -d #to create this wallet
if err != nil {
  log.Fatal(err)
}
acct, err := wlt.GetDefaultAccount([]byte("walletpasswd"))
if err != nil {
  log.Fatal(err)
}
did := "did:ont:" + acct.Address.ToBase58()
// register did
tx, err := sdk.Native.OntId.RegIDWithPublicKey(2500, 20000, acct, did, acct)
if err != nil {
  log.Fatal(err)
}
log.Printf("txhash: %s", tx.ToHexString())
```
Please refer to below links for details on this process:

- Create a DID following this [instruction](https://github.com/ontio/ontology-go-sdk#25-ont-id-api).
- Create an Ontology wallet following this [instruction](https://github.com/ontio/ontology/blob/master/docs/specifications/cli_user_guide.md#2-wallet-management).
- Apply for testnet ONG for testing [here](https://developer.ont.io/applyOng).

# Request Format

All API requests from the DP should contain two additional fields: 

| Field        | Type     | Description                                                                                                                    |
| ------------ | -------- | ------------------------------------------------------------------------------------------------------------------------------ |
| `caller_did` | `string` | caller DID                                                                                                                     |
| `encrypt`    | `bool`   | true -> the response should be encrypted using the associated publicKey of the caller DID<br>false -> response with plain text |

    
Example of requesting an unencrypted response:
    
```go
{
  "caller_did": "did:ont:AU7YUFqUcbNC7baGZ2Ei4Biy5sUD7GLr2X",
  "encrypt": false,
  "requestData":{
      	<original request data>
  }	
}
```
    
Example of requesting an encrypted response:

```go    
{
  "caller_did": "did:ont:AU7YUFqUcbNC7baGZ2Ei4Biy5sUD7GLr2X",
  "encrypt": true,
  "requestData":{
  	<original request data>
  }	
}
```


# Response Format

## Unencrypted Response

The response consists of two parts:

-  `data` object containing below fields:
  
| Field  | Type     | Description                                       |
| ------ | -------- | ------------------------------------------------- |
| `data` | `object` | BrightID response data                            |
| `sign` | `string` | signature for the response data with the node DID |

- `encrypt` with the value `null` 
    
Example:

```go
{
"data": {
  "data": {...},
  "sign": "abeca32...",
},
"encrypt": null
}
```

## Encrypted Response

The response is the hex string of the serialized `data` object being encrypted.

Example:

```go
{
  "encrypt": "eac4aadc..." // hex format
}
```

Process of encryption using the DID (in Go):

```go   
import (
  "crypto/elliptic"
  "crypto/rand"
  "encoding/hex"
  "encoding/json"
  "errors"

  "github.com/ethereum/go-ethereum/crypto/ecies"
  "github.com/ontio/ontology-crypto/ec"
  ontsdk "github.com/ontio/ontology-go-sdk"
)
    
type DidPubkey struct {
  Id           string      `json:"id"`
  Type         string      `json:"type"`
  Controller   interface{} `json:"controller"`
  PublicKeyHex string      `json:"publicKeyHex"`
}
    
func (ss *Service) EncryptDataWithDID(input []byte, did string) ([]byte, error) {
  sdk := ontsdk.NewOntologySdk()
  sdk.NewRpcClient().SetAddress(ss.OntologyRPCURL)// testnet: http://polaris2.ont.io:20336, mainnet: http://dappnode2.ont.io:20336
  dpubb, err := sdk.Native.OntId.GetPublicKeysJson(did)
  if err != nil {
    return nil, err
  }
    
  var pks DidPubkey
  err = json.Unmarshal(dpubb, &pks)
  if err != nil {
    return nil, err
  }
  if len(pks) == 0 {
    return nil, errors.New("pubkey not found")
  }
  pubb, err := hex.DecodeString(pks[0].PublicKeyHex)
  if err != nil {
    return nil, err
  }
    
  pub, err := ec.DecodePublicKey(pubb, elliptic.P256())
  if err != nil {
    return nil, err
  }
  epub := ecies.ImportECDSAPublic(pub)

  return ecies.Encrypt(rand.Reader, epub, input, nil, nil)
}
```
