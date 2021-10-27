# vault_encrypt

Requiriments do start

OCI Console Web:

- create a Vault (not private)
- create a Master Encryption Key with Protection Mode HSM, algoritm AES and 256 bits

**Its important to create policies in OCI for a only group can manipulate and manage secrets, vaults and keys**

Follow the example

```bash
allow ... to manage secret-family in tenancy
allow ... to manage vaults in tenancy
allow ... to manage keys in tenancy
```

Below its a example to **create a bucket using the Oracle Vault**. 
The script load the setting file from local machine, but you can change this. Its up to you. 

```python
import oci
import sys

config = oci.config.from_file()
object_storage_client = oci.object_storage.ObjectStorageClient(config)

namespace = object_storage_client.get_namespace().data
compartment_id = "ocid1.tenancy.oc1......"
bucket_name = "meu_querido_bucket"
kms_oci_id = "ocid1.key.oc1.iad.bbp2....."

"""
O tipo de acesso poderá ser: NoPublicAccess, ObjectRead, ObjectReadWithoutList
O tipo de storage poderá ser: Archive ou Standart
"""

object_details = oci.object_storage.models.CreateBucketDetails(
    name=bucket_name, 
    storage_tier='Standard', 
    compartment_id=compartment_id, 
    public_access_type='ObjectRead', 
    kms_key_id=kms_oci_id)

create_bucket_response = object_storage_client.create_bucket(namespace, object_details)

```

Below its the script to encryt the content file and upload to object. Also has a function to download and decryp the content. Replace the ID in Vault and Master Key.

```python
import psutil, oci, os, shutil, time
from oci import auth, identity, encryption
 
config = oci.config.from_file()
object_storage_client = oci.object_storage.ObjectStorageClient(config)

file_path = 'teste.txt'
bucket = "meu_querido_bucket"

kms_master_key = encryption.KMSMasterKey(
    config=config,
    vault_id="ocid1.vault.....", 
    master_key_id="ocid1.key...." 
)
 
kms_provider = encryption.KMSMasterKeyProvider(
      config=config, 
      kms_master_keys=[kms_master_key])

namespace = object_storage_client.get_namespace().data

def encrypt_file(path):
    with open(path + '.enc', 'wb') as output_stream, open(path, 'rb') as input_file:
        with oci.encryption.create_encryption_stream(
            master_key_provider=kms_provider,
            stream_to_encrypt=input_file,
        ) as encryption_stream:
            shutil.copyfileobj(encryption_stream, output_stream) #salvei local p teste

def download_item(path):
    resp = object_storage_client.get_object(
        namespace_name=namespace, 
        bucket_name=bucket, 
        object_name=path + ".enc"
    )
    with oci.encryption.create_decryption_stream(
        master_key_provider=kms_provider,
        stream_to_decrypt=resp.data.raw
        ) as decryption_stream:
            decrypted_content = decryption_stream.read()
            print(decrypted_content)

def upload_item(path):
    encrypt_file(file_path)
    with open(path + '.enc', "rb") as in_file:
        name = os.path.basename(path + '.enc')
        objStgClient = oci.object_storage.ObjectStorageClient(config)
        resp = objStgClient.put_object(
            namespace_name=namespace, 
            bucket_name=bucket, 
            object_name=name, 
            put_object_body=in_file, 
            opc_sse_kms_key_id='ocid1.....ew6...a'
        )

upload_item(file_path)
download_item(file_path)
```

When you access the content from bucket you get a file like this:

```json
{"encryptedContentFormat": 0, "encryptedDataKeys": 
[{"masterKeyId": "ocid1.key.oc1........", 
  "vaultId": "ocid1.va....", 
  "encryptedDataKey": "QVW2vkWrGT....IKWOFiOiiQ0h7xhpT8vP2XIoyxNAQK/SUfdNAAAAAA=", 
  "region": "us-ashburn-1"}], 
  "iv": "6/2NmUpl2citiDVo", 
  "algorithmId": 0, 
  "additionalAuthenticatedData": ""}
```

❤️ fernando.d.costa@oracle.com
