# Transitioner
This function generates a proof of transition from the old merkle root to the new merkle root.

# Common Types

## Input 

An input of a merkle state transition. 

```protobuf
message Input {
    bytes txRaw = 1;
    bytes signedTxRaw = 2;
    bytes senderMerkleProof = 3;
    bytes recieverMerkleProof = 4;
    bytes currentMerkleRoot = 5;
}
```

## Output 
An output of a merkle state transtion. 
```protobuf
message Output {
    bytes newMerkleRoot = 1;
    bytes proof = 2;
}
```