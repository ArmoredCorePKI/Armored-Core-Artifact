# Armored-Core How to Run

Artifact guidance for the paper *Armored Core of PKI: Removing Signing Keys for CA via Efficient and Trusted Physical Certification*

[Arxiv](https://arxiv.org/abs/2404.15582) | [PapersWithCode](https://paperswithcode.com/paper/armored-core-of-pki-remove-signing-keys-for) | [Code Repos](https://github.com/ArmoredCorePKI)

<img title="" src="figs/acorelogo-nobg.png" alt="" align="center" data-align="center">

> This page is not finished, the docker environment is in preparation.

## 🔨 Prerequisites

- Make sure you have the `sudo` and `root` privileges for your machine
- Golang environment (verision >= 1.21)

## 📁 Component

- 🏢 **CA** (on Lets'Encrypt [Pebble](https://github.com/letsencrypt/pebble)): https://github.com/ArmoredCorePKI/Armored-Core-Pebble
- 📒 **Logger** (on Google [Trillian](https://github.com/google/trillian)): We can directly use the original Trillian codebase https://github.com/google/trillian 
- 🌐 **Domain** (on [Certbot](https://github.com/certbot/certbot)): https://github.com/ArmoredCorePKI/Armored-Core-Certbot
- 🖥️ **Client** (Mozilia Web Extension): https://github.com/ArmoredCorePKI/Armored-Core-Extension

## ⚙️ Manually Run the system

- Step1: Prepare four terminals
  
  - 🔷 one terminal for Logger
  - 🔶 one terminal for CA
  - 🔴 one terminal for Domain
  - ⚫ one root terminal

- Step2: Configure the local domain on the server

```bash
sudo vim /etc/hosts

# add records: 127.0.0.1 mydomain.test`
```

- (⚫) Remove the old Let's Encrypt File 

```bash
rm -rf /etc/letsencrypt/archive/mydomain.test && rm -rf /etc/letsencrypt/live/mydomain.test && rm -rf /etc/letsencrypt/renewal/mydomain.test.conf && rm -rf /etc/letsencrypt/accounts/localhost\:14000/
```

- Step3: (🔷) Run the trillian log server 

```bash
go run github.com/google/trillian/cmd/trillian_log_server --rpc_endpoint="localhost:50054" --http_endpoint="localhost:50055"

go run github.com/google/trillian/cmd/trillian_log_signer --sequencer_interval="1s" --batch_size=500 --rpc_endpoint="localhost:50056" --http_endpoint="localhost:50057" --num_sequencers=1 --force_master
```

Remember the Tree ID, e.g., 84734734810528304839

- Step4: (🔶) Run Armored Core Pebble 

```bash
cd project_dir
go install ./cmd/pebble
~/go/bin/pebble -config ./test/config/pebble-config.json -treeid=the_tree_id
```

- Step5: (🔴) Certbot request and verify certificates 

```bash
cd project_dir/certbot
sudo your_virtual_python3_bin main.py certonly --standalone -d mydomain.test --server https://localhost:14000/dir --email you@example.com --agree-tos --no-verify-ssl --http-01-port=5002
```

> We now use the root terminal (⚫) to request the root certificate
> `curl -k -s -o /etc/letsencrypt/live/mydomain.test/roots.pem https://localhost:15000/roots/0`

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

## :whale: Docker

> The dockerfile and docker compose file are not ready yet and we first demonstrate the following instructions.

-  Compiling the three containers
`docker-compose up --build --force-recreate`
- You can re-built them without cache `docker-compose build --no-cache`

- Then acquire the ip address of the three containers for later use.
`docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}} {{end}}' acore-ca acore-domain acore-logger`

- Next, attach into the containers to execute the Armored PKI system.
- First, attach into the logger container 
	- execute the init script `sh ./init.sh`, the password of mysql is null, just directly press Enter.
	- You may see some connection refused errors in the beginning but that's okay. In the end, the TREE ID should be printed out in the terminal.
	- We can check the trillian service by `ps -aux | grep trillian`, which should show the corresponding log server and log signer process.

- Second, attach to the CA container. We can forget about the logger container for now.
	- `vim /etc/hosts`, then add `the_domain_container ip	mydomain.acore`
	- `go install --buildvcs=false ./cmd/pebble`
	- `~/go/bin/pebble -config ./test/config/pebble-config.json -treeid=YOUR_TREE_ID`

- At last, attach into the Domain container.
	- `cd ./certbot`
	- `python3 main.py certonly --standalone -d mydomain.acore --server https://acore-ca:14000/dir --email you@example.com --agree-tos --no-verify-ssl --http-01-port=5002`

- Then certificate issuance is completed. We can check out the output and logs in the terminals from the three containers.


## 👀 Evaluation

TODO

## 🙇‍♂️ Acknowledgements

We appreciate the following projects.

- [Pebble](https://github.com/letsencrypt/pebble)
- [Trillian](https://github.com/google/trillian)
- [Certificate Transparency](https://github.com/google/certificate-transparency-go)
- [Certbot](https://github.com/certbot/certbot)
- [F-PKI](https://github.com/netsec-ethz/fpki) (NDSS' 2022) 
