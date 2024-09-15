# Armored-Core How to Run
> This page is not finished, but the general procedure is presented as below.
> 
## Component

- CA (on Lets'Encrypt Pebble): https://github.com/ArmoredCorePKI/Armored-Core-Pebble
- Log server (on Google Trillian): We can directly use the original Trillian codebase https://github.com/google/trillian 
- Domain (on Certbot): https://github.com/ArmoredCorePKI/Armored-Core-Certbot
- Client (Mozilia Web Extension): https://github.com/ArmoredCorePKI/Armored-Core-Extension

## Prerequisite

- Make sure you have the `sudo` and `root` privileges for your machine
- Go 

## Run the system

- Step1: Prepare four terminals
  - (TR) one terminal for trillian server
  - (PB) one terminal for pebble
  - (CB) one terminal for certbot
  - (RT) one root terminal

- Step2: Configure local domain

```bash
sudo vim /etc/hosts

# add records: 127.0.0.1 mydomain.test`
```
- Remove the old Let's Encrypt File (RT)

```bash
rm -rf /etc/letsencrypt/archive/mydomain.test && rm -rf /etc/letsencrypt/live/mydomain.test && rm -rf /etc/letsencrypt/renewal/mydomain.test.conf && rm -rf /etc/letsencrypt/accounts/localhost\:14000/
```

- Step3: Run the trillian log server (TR)

```bash
go run github.com/google/trillian/cmd/trillian_log_server --rpc_endpoint="localhost:50054" --http_endpoint="localhost:50055"

go run github.com/google/trillian/cmd/trillian_log_signer --sequencer_interval="1s" --batch_size=500 --rpc_endpoint="localhost:50056" --http_endpoint="localhost:50057" --num_sequencers=1 --force_master
```

Remember the Tree ID.


- Step4: Run Armored Core Pebble (PB)

```bash
cd project_dir
go install ./cmd/pebble
~/go/bin/pebble -config ./test/config/pebble-config.json -treeid=the_tree_id
```

- Step5: Certbot request and verify certificates (CB)

```bash
cd project_dir/certbot
sudo your_virtual_python3_bin main.py certonly --standalone -d mydomain.test --server https://localhost:14000/dir --email you@example.com --agree-tos --no-verify-ssl --http-01-port=5002
```

> We now use the RT to request the root certificate
`curl -k -s -o /etc/letsencrypt/live/mydomain.test/roots.pem https://localhost:15000/roots/0`

Then we can verify the whole certificate chain and entry chain. (make sure first edit the `BASEINDEX` in `trillianclient/tclient_util.pp` as the AppendEntry `treesize - 4`, we will improve this workflow later :-) )

```bash
sudo your_virtual_python3_bin main.py main.py acorefunc --verify-puf-cert --test-domain mydomain.test
```

- Step6: Client Extension verify certificates

```bash
cd project_dir
web-ext run
```

You can open the Browser console and see the output.
