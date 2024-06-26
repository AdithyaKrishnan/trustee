# KBS Cluster

KBS provides a simple cluster defined by `docker-compose`, include itself, [Attestation Service](https://github.com/confidential-containers/trustee/tree/main/attestation-service), [Reference Value Provider Service](https://github.com/confidential-containers/trustee/tree/main/attestation-service/rvps) and [CoCo Keyprovider](https://github.com/confidential-containers/guest-components/tree/main/attestation-agent/coco_keyprovider)

Users can use very simple command to:
- launch KBS service.
- encrypt images.

## Architecture

<div align=center>

![](./pictures/cluster.svg)

</div>

## Start-Up

Generate a user auth key pair
```
cd ./trustee/KBS
openssl genpkey -algorithm ed25519 > ./config/private.key
openssl pkey -in ./config/private.key -pubout -out ./config/public.pub
```

Uninstall old versions of docker
```bash
# Run the following command to uninstall all conflicting packages:
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done
```

Install latest version of docker using the apt repository.

Set up Docker's apt repository.
```bash
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
```

Install the latest version of docker
```bash
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Run the cluster
```bash
docker compose up -d
```

Note that by default the KBS cluster blocks sample evidence.
If you are testing with sample evidence you will need to
set a more permissive resource policy.

Then the kbs cluster is launched.

Use `skopeo` to encrypt an image
```bash
# edit ocicrypt.conf
tee > ocicrypt.conf <<EOF
{
    "key-providers": {
        "attestation-agent": {
            "grpc": "127.0.0.1:50000"
        }
    }
}
EOF

# encrypt the image
OCICRYPT_KEYPROVIDER_CONFIG=ocicrypt.conf skopeo copy --insecure-policy --encryption-key provider:attestation-agent docker://busybox oci:busybox_encrypted
```

The image will be encrypted, and things happens in the background include:
- `CoCo Keyprovider` generates a random KEK and a key id. Then encrypts the image using the KEK.
- `CoCo Keyprovider` registers the KEK with key id into KBS.

If use the same KBS for key brokering, the image can be decrypted.
