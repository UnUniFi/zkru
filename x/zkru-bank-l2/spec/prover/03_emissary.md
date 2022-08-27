# Emissary 
This module make a proof of current consensus state. 

# Common Types

```protobuf
message blockHeader {
    int32 height 
```

## Input

``` protobuf
message Input {
    blockHeader blockheader = 1;
    bytes proof = 1;
}
``` 

## Output 

``` protobuf
message Output {
    bytes proof = 1;
}
```