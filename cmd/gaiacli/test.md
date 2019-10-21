# IBC Test

**Environment setup**

```bash
cd ~ && mkdir ibc-testnets && cd ibc-testnets
gaiad testnet -o ibc-a --v 1 --chain-id chain-a --node-dir-prefix n
gaiad testnet -o ibc-b --v 1 --chain-id chain-b --node-dir-prefix n
```

**Set configuration**

```bash
sed -i '' 's/"leveldb"/"goleveldb"/g' ibc-a/n0/gaiad/config/config.toml
sed -i '' 's/"leveldb"/"goleveldb"/g' ibc-b/n0/gaiad/config/config.toml
sed -i '' 's#"tcp://0.0.0.0:26656"#"tcp://0.0.0.0:26556"#g' ibc-b/n0/gaiad/config/config.toml
sed -i '' 's#"tcp://0.0.0.0:26657"#"tcp://0.0.0.0:26557"#g' ibc-b/n0/gaiad/config/config.toml
sed -i '' 's#"localhost:6060"#"localhost:6061"#g' ibc-b/n0/gaiad/config/config.toml
sed -i '' 's#"tcp://127.0.0.1:26658"#"tcp://127.0.0.1:26558"#g' ibc-b/n0/gaiad/config/config.toml

gaiacli config --home ibc-a/n0/gaiacli/ chain-id chain-a
gaiacli config --home ibc-b/n0/gaiacli/ chain-id chain-b
gaiacli config --home ibc-a/n0/gaiacli/ output json
gaiacli config --home ibc-b/n0/gaiacli/ output json
gaiacli config --home ibc-a/n0/gaiacli/ node http://localhost:26657
gaiacli config --home ibc-b/n0/gaiacli/ node http://localhost:26557
```

**View Keys**

```bash
gaiacli --home ibc-a/n0/gaiacli keys list | jq '.[].address'
gaiacli --home ibc-b/n0/gaiacli keys list | jq '.[].address'
```

**Start**

```bash
# run in background
nohup gaiad --home ibc-a/n0/gaiad start >ibc-a.log &
nohup gaiad --home ibc-b/n0/gaiad start >ibc-b.log &

# run in terminal
gaiad --home ibc-a/n0/gaiad start
gaiad --home ibc-b/n0/gaiad start
```

**Create client**

create client on chain-a

```bash
# view consensus-state of chain-b
gaiacli --home ibc-b/n0/gaiacli q ibc client self-consensus-state -o json | jq
# export consensus_state.json from chain-b
gaiacli --home ibc-b/n0/gaiacli q ibc client self-consensus-state -o json >ibc-a/n0/consensus_state.json
# create client on chain-a
gaiacli --home ibc-a/n0/gaiacli tx ibc client create client-to-b ibc-a/n0/consensus_state.json \
  --from n0 -y -o text --broadcast-mode=block
```

create client on chain-b

```bash
# view consensus-state of chain-a
gaiacli --home ibc-a/n0/gaiacli q ibc client self-consensus-state -o json | jq
# export consensus_state.json from chain-a
gaiacli --home ibc-a/n0/gaiacli q ibc client self-consensus-state -o json >ibc-b/n0/consensus_state.json
# create client on chain-b
gaiacli --home ibc-b/n0/gaiacli tx ibc client create client-to-a ibc-b/n0/consensus_state.json \
  --from n0 -y -o text --broadcast-mode=block
```

**Query client**

query client state

```bash
# query client state on chain-a
gaiacli --home ibc-a/n0/gaiacli q ibc client state client-to-b | jq
# query client state on chain-b
gaiacli --home ibc-b/n0/gaiacli q ibc client state client-to-a | jq
```

query client consensus-state

```bash
# query client consensus-state on chain-a
gaiacli --home ibc-a/n0/gaiacli q ibc client consensus-state client-to-b | jq
# query client consensus-state on chain-b
gaiacli --home ibc-b/n0/gaiacli q ibc client consensus-state client-to-a | jq
```

query client path

```bash
# query client path of chain-a
gaiacli --home ibc-a/n0/gaiacli q ibc client path | jq
# query client path of chain-b
gaiacli --home ibc-b/n0/gaiacli q ibc client path | jq
```

**Update client**

update chain-a

```bash
# query header of chain-b
gaiacli --home ibc-b/n0/gaiacli q ibc client header -o json | jq
# export header of chain-b
gaiacli --home ibc-b/n0/gaiacli q ibc client header -o json >ibc-a/n0/header.json
# update client on chain-a
gaiacli --home ibc-a/n0/gaiacli tx ibc client update client-to-b ibc-a/n0/header.json \
  --from n0 -y -o text --broadcast-mode=block
```

update chain-b

```bash
# query header of chain-a
gaiacli --home ibc-a/n0/gaiacli q ibc client header -o json | jq
# export header of chain-a
gaiacli --home ibc-a/n0/gaiacli q ibc client header -o json >ibc-b/n0/header.json
# update client on chain-b
gaiacli --home ibc-b/n0/gaiacli tx ibc client update client-to-a ibc-b/n0/header.json \
  --from n0 -y -o text --broadcast-mode=block
```

**Create connection**

`open-init` on chain-a

```bash
# export prefix.json
gaiacli --home ibc-b/n0/gaiacli q ibc client path -o json >ibc-a/n0/prefix.json
# view prefix.json
jq -r '' ibc-a/n0/prefix.json
# open-init
gaiacli --home ibc-a/n0/gaiacli tx ibc connection open-init \
  conn-to-b client-to-b \
  conn-to-a client-to-a \
  ibc-a/n0/prefix.json \
  --from n0 -y -o text \
  --broadcast-mode=block
```

`open-try` on chain-b

```bash
# export prefix.json
gaiacli --home ibc-a/n0/gaiacli q ibc client path -o json >ibc-b/n0/prefix.json
# export header.json from chain-a
gaiacli --home ibc-a/n0/gaiacli q ibc client header -o json >ibc-b/n0/header.json
# export proof_init.json from chain-a with hight in header.json
gaiacli --home ibc-a/n0/gaiacli q ibc connection proof conn-to-b \
  $(jq -r '.value.SignedHeader.header.height' ibc-b/n0/header.json) \
  -o json >ibc-b/n0/conn_proof_init.json
# view proof_init.json
jq -r '' ibc-b/n0/conn_proof_init.json
# update client on chain-b
gaiacli --home ibc-b/n0/gaiacli tx ibc client update client-to-a ibc-b/n0/header.json \
  --from n0 -y -o text --broadcast-mode=block
# query client consense state
gaiacli --home ibc-b/n0/gaiacli q ibc client consensus-state client-to-a | jq
# open-try
gaiacli --home ibc-b/n0/gaiacli tx ibc connection open-try \
  conn-to-a client-to-a \
  conn-to-b client-to-b \
  ibc-b/n0/prefix.json \
  1.0.0 \
  ibc-b/n0/conn_proof_init.json \
  $(jq -r '.value.SignedHeader.header.height' ibc-b/n0/header.json) \
  --from n0 -y -o text \
  --broadcast-mode=block
```

`open-ack` on chain-a

```bash
# export header.json from chain-b
gaiacli --home ibc-b/n0/gaiacli q ibc client header -o json >ibc-a/n0/header.json
# export proof_try.json from chain-b with hight in header.json
gaiacli --home ibc-b/n0/gaiacli q ibc connection proof conn-to-a \
  $(jq -r '.value.SignedHeader.header.height' ibc-a/n0/header.json) \
  -o json >ibc-a/n0/conn_proof_try.json
# view proof_try.json
jq -r '' ibc-a/n0/conn_proof_try.json
# update client on chain-a
gaiacli --home ibc-a/n0/gaiacli tx ibc client update client-to-b ibc-a/n0/header.json \
  --from n0 -y -o text --broadcast-mode=block
# query client consense state
gaiacli --home ibc-a/n0/gaiacli q ibc client consensus-state client-to-b | jq
# open-ack
gaiacli --home ibc-a/n0/gaiacli tx ibc connection open-ack \
  conn-to-b \
  ibc-a/n0/conn_proof_try.json \
  $(jq -r '.value.SignedHeader.header.height' ibc-a/n0/header.json) \
  1.0.0 \
  --from n0 -y -o text \
  --broadcast-mode=block
```

`open-confirm` on chain-b

```bash
# export header.json from chain-a
gaiacli --home ibc-a/n0/gaiacli q ibc client header -o json >ibc-b/n0/header.json
# export proof_ack.json from chain-a with hight in header.json
gaiacli --home ibc-a/n0/gaiacli q ibc connection proof conn-to-b \
  $(jq -r '.value.SignedHeader.header.height' ibc-b/n0/header.json) \
  -o json >ibc-b/n0/conn_proof_ack.json
# view proof_ack.json
jq -r '' ibc-b/n0/conn_proof_ack.json
# update client on chain-b
gaiacli --home ibc-b/n0/gaiacli tx ibc client update client-to-a ibc-b/n0/header.json \
  --from n0 -y -o text --broadcast-mode=block
# open-confirm
gaiacli --home ibc-b/n0/gaiacli tx ibc connection open-confirm \
  conn-to-a \
  ibc-b/n0/conn_proof_ack.json \
  --from n0 -y -o text \
  $(jq -r '.value.SignedHeader.header.height' ibc-b/n0/header.json) \
  --broadcast-mode=block
```

**Query connection**

query connection

```bash
# query connection on chain-a
gaiacli --home ibc-a/n0/gaiacli q ibc connection end conn-to-b | jq
# query connection on chain-b
gaiacli --home ibc-b/n0/gaiacli q ibc connection end conn-to-a | jq
```

query connection proof

```bash
# query connection proof with height in header.json on chain-a
gaiacli --home ibc-a/n0/gaiacli q ibc connection proof conn-to-b \
  $(jq -r '.value.SignedHeader.header.height' ibc-a/n0/header.json) | jq
# query connection proof with height in header.json on chain-b
gaiacli --home ibc-b/n0/gaiacli q ibc connection proof conn-to-a \
  $(jq -r '.value.SignedHeader.header.height' ibc-b/n0/header.json) | jq
```

query connections of a client

```bash
# query connections of a client on chain-a
gaiacli --home ibc-a/n0/gaiacli q ibc connection client client-to-b | jq
# query connections of a client on chain-b
gaiacli --home ibc-b/n0/gaiacli q ibc connection client client-to-a | jq
```

**Create channel**

`open-init` on chain-a

```bash
# open-init
gaiacli --home ibc-a/n0/gaiacli tx ibc channel open-init \
  port-to-bank chann-to-b \
  port-to-bank chann-to-a \
  conn-to-b \
  --from n0 -y -o text \
  --broadcast-mode=block
```

`open-try` on chain-b

```bash
# export header.json from chain-a
gaiacli --home ibc-a/n0/gaiacli q ibc client header -o json >ibc-b/n0/header.json
# export proof_init.json from chain-a with hight in header.json
gaiacli --home ibc-a/n0/gaiacli q ibc channel proof port-to-bank chann-to-b \
  $(jq -r '.value.SignedHeader.header.height' ibc-b/n0/header.json) \
  -o json >ibc-b/n0/chann_proof_init.json
# view proof_init.json
jq -r '' ibc-b/n0/chann_proof_init.json
# update client on chain-b
gaiacli --home ibc-b/n0/gaiacli tx ibc client update client-to-a ibc-b/n0/header.json \
  --from n0 -y -o text --broadcast-mode=block
# query client consense state
gaiacli --home ibc-b/n0/gaiacli q ibc client consensus-state client-to-a | jq
# open-try
gaiacli --home ibc-b/n0/gaiacli tx ibc channel open-try \
  port-to-bank chann-to-a \
  port-to-bank chann-to-b \
  conn-to-a \
  ibc-b/n0/chann_proof_init.json \
  $(jq -r '.value.SignedHeader.header.height' ibc-b/n0/header.json) \
  --from n0 -y -o text \
  --broadcast-mode=block
```

`open-ack` on chain-a

```bash
# export header.json from chain-b
gaiacli --home ibc-b/n0/gaiacli q ibc client header -o json >ibc-a/n0/header.json
# export proof_try.json from chain-b with hight in header.json
gaiacli --home ibc-b/n0/gaiacli q ibc channel proof port-to-bank chann-to-a \
  $(jq -r '.value.SignedHeader.header.height' ibc-a/n0/header.json) \
  -o json >ibc-a/n0/chann_proof_try.json
# view proof_try.json
jq -r '' ibc-a/n0/chann_proof_try.json
# update client on chain-a
gaiacli --home ibc-a/n0/gaiacli tx ibc client update client-to-b ibc-a/n0/header.json \
  --from n0 -y -o text --broadcast-mode=block
# query client consense state
gaiacli --home ibc-a/n0/gaiacli q ibc client consensus-state client-to-b | jq
# open-ack
gaiacli --home ibc-a/n0/gaiacli tx ibc channel open-ack \
  port-to-bank chann-to-b \
  ibc-a/n0/chann_proof_try.json \
  $(jq -r '.value.SignedHeader.header.height' ibc-a/n0/header.json) \
  --from n0 -y -o text \
  --broadcast-mode=block
```

`open-confirm` on chain-b

```bash
# export header.json from chain-a
gaiacli --home ibc-a/n0/gaiacli q ibc client header -o json >ibc-b/n0/header.json
# export proof_ack.json from chain-a with hight in header.json
gaiacli --home ibc-a/n0/gaiacli q ibc channel proof port-to-bank chann-to-b \
  $(jq -r '.value.SignedHeader.header.height' ibc-b/n0/header.json) \
  -o json >ibc-b/n0/chann_proof_ack.json
# view proof_ack.json
jq -r '' ibc-b/n0/chann_proof_ack.json
# update client on chain-b
gaiacli --home ibc-b/n0/gaiacli tx ibc client update client-to-a ibc-b/n0/header.json \
  --from n0 -y -o text --broadcast-mode=block
# query client consense state
gaiacli --home ibc-b/n0/gaiacli q ibc client consensus-state client-to-a | jq
# open-confirm
gaiacli --home ibc-b/n0/gaiacli tx ibc channel open-confirm \
  port-to-bank chann-to-a \
  ibc-b/n0/chann_proof_ack.json \
  $(jq -r '.value.SignedHeader.header.height' ibc-b/n0/header.json) \
  --from n0 -y -o text \
  --broadcast-mode=block
```

**Query channel**

query channel

```bash
# query channel on chain-a
gaiacli --home ibc-a/n0/gaiacli query ibc channel end port-to-bank chann-to-b | jq
# query channel on chain-b
gaiacli --home ibc-b/n0/gaiacli query ibc channel end port-to-bank chann-to-a | jq
```

query channel proof

```bash
# query channel proof with height in header.json on chain-a
gaiacli --home ibc-a/n0/gaiacli q ibc channel proof port-to-bank chann-to-b \
  $(jq -r '.value.SignedHeader.header.height' ibc-a/n0/header.json) | jq
# query channel proof with height in header.json on chain-b
gaiacli --home ibc-b/n0/gaiacli q ibc channel proof port-to-bank chann-to-a \
  $(jq -r '.value.SignedHeader.header.height' ibc-b/n0/header.json) | jq
```

**Bank transfer from chain-a to chain-b**

```bash
# export transfer result to result.json
gaiacli --home ibc-a/n0/gaiacli tx ibcmockbank transfer \
  --src-port port-to-bank --src-channel chann-to-b \
  --denom uiris --amount 1 \
  --receiver $(gaiacli --home ibc-b/n0/gaiacli keys show n0 | jq -r '.address') \
  --source true \
  --from n0 -y -o json > ibc-a/n0/result.json
# export packet.json
jq -r '.events[1].attributes[2].value' ibc-a/n0/result.json >ibc-b/n0/packet.json
```

**Bank receive**

> use "tx ibc channel send-packet" instead when router-module completed in the future

```bash
# export header.json from chain-a
gaiacli --home ibc-a/n0/gaiacli q ibc client header -o json >ibc-b/n0/header.json
# export proof.json from chain-b with hight in header.json
gaiacli --home ibc-a/n0/gaiacli q ibc channel proof port-to-bank chann-to-b \
  $(jq -r '.value.SignedHeader.header.height' ibc-b/n0/header.json) \
  -o json >ibc-b/n0/proof.json
# view proof.json
jq -r '' ibc-b/n0/proof.json
# update client on chain-b
gaiacli --home ibc-b/n0/gaiacli tx ibc client update client-to-a ibc-b/n0/header.json \
  --from n0 -y -o text --broadcast-mode=block
# receive packet
gaiacli --home ibc-b/n0/gaiacli tx ibcmockbank recv-packet \
  ibc-b/n0/packet.json ibc-b/n0/proof.json \
  $(jq -r '.value.SignedHeader.header.height' ibc-b/n0/header.json) \
  --from n0 -y -o text \
  --broadcast-mode=block
```

**Query Account**

```bash
# view sender account
gaiacli --home ibc-a/n0/gaiacli q account -o text \
  $(gaiacli --home ibc-a/n0/gaiacli keys show n0 | jq -r '.address')
# view receiver account
gaiacli --home ibc-b/n0/gaiacli q account -o text \
  $(gaiacli --home ibc-b/n0/gaiacli keys show n0 | jq -r '.address')
```
