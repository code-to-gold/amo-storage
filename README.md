# AMO Storage Service

## Introduction
Service for storing data which is traded on AMO Blockchain
- Support REST API for operations related to CRUD of data
- Support authentication based on client's identity in AMO blockchain
- Support authorization through communication with AMO blockchain

## Quick Start
### From Docker
#### Pre-requisites
- [Docker](https://docs.docker.com/v17.12/docker-for-mac/install/) (higher than version 17)
- [Docker Compose](https://docs.docker.com/compose/install/)


#### Configurations
Make your own config directory and put `config.ini` and `key.json` into the directory.
Example of config.ini and key.json file is in the `./config` directory.
You must set `REDIS_HOST` as `redis` like below.

```ini
; config.ini example
[AmoStorageConfig]
; App configurations
DEBUG=0

; SQLite Configurations
SQLALCHEMY_TRACK_MODIFICATIONS=0
SQLALCHEMY_DATABASE_URI=sqlite:////tmp/amo_storage.db

; Redis Configurations
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_DB=0

; Service ID for data parcel
SERVICE_ID=00000001

[AuthConfig]
ISSUER=amo-storage
ALGORITHM=HS256
SECRET=your-sercret

[CephConfig]
HOST=127.0.0.1
PORT=7480
BUCKET_NAME=amo

[AmoBlockchainNodeConfig]
HOST=127.0.0.1
PORT=26657
```

##### Setting key file for CEPH authentication
***NOTE:*** *This section is not for the CEPH's configuration or configuring the CEPH cluster itself but for connecting to existing CEPH properly.* 
***It is assumed that the CEPH cluster is constructed and configured separately*** and
***it is assumed that there exists a CEPH Rados Gateway running.***

- The `access_key` and `secret_key` which are used for connecting to the **Rados Gateway** can be found in the `key` attribute of CEPH user.
(We assume there is a Rados Gateway user `amoapi` for this adapter software. This may become configurable in the future version of this software.)
- The `access_key` and `secret_key` should be included in the `key.json` file.
```son
{
    "access_key":"{USER'S_ACCESS_KEY}",
    "secret_key":"{USER'S_SECRET_KEY}"
}
```

Set `CONFIG_DIR` as your own config directory and set `PORT` as storage service going to listen.
```bash
$ export CONFIG_DIR=/tmp
$ export PORT=5000
```
Or you can create `.env` at the same path of docker-compose.yml
```
// .env example
CONFIG_DIR=/tmp
PORT=5000
```
#### Run
Run below command at the same path of docker-compose.yml
```bash
$ docker-compose up -d
```


### From Source
#### Pre-requisites
- [Python3](https://www.python.org/downloads/release/python-368/) (compatible with 3.6.8)
- Python3-pip (compatible with 9.0.1)
- [Redis](https://redis.io/download) (compatible with 5.0.5)
- [Sqlite](https://www.sqlite.org/download.html) (compatible with 5.0.5)
	

#### Configurations
Config directory should contain `config.ini` and `key.json`. The default configurations directory is provided with the name `config` but you can specify the config directory path manually. 

Most of the configurations can be set in `config.ini` file except access key and secret used for connecting to CEPH.
Below is an example of `config.ini`
You should set configurations for your environment.
```ini
[AmoStorageConfig]
; App configurations
DEBUG=0

; SQLite Configurations
SQLALCHEMY_TRACK_MODIFICATIONS=0
SQLALCHEMY_DATABASE_URI=sqlite:////tmp/amo_storage.db

; Redis Configurations
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_DB=0

; Service ID for data parcel
SERVICE_ID=00000001

[AuthConfig]
ISSUER=amo-storage
ALGORITHM=HS256
SECRET=your-sercret

[CephConfig]
HOST=127.0.0.1
PORT=7480
BUCKET_NAME=amo

[AmoBlockchainNodeConfig]
HOST=127.0.0.1
PORT=26657
```

To get more information about `key.json`, read [this section](#setting-key-file-for-ceph-authentication)


#### Run
***NOTE:*** *Before starting AMO-Storage API server, `redis` server daemon must be running.*

To run AMO-Storage API server, some dependencies must be installed. Install via below command.
```shell
$ pip3 install -r requirements.txt
```

And then run amo-stroage service via below command.
```shell
$ python3 main.py
* running on http://127.0.0.1:5000
```

Also you can run amo-storage service on specific host, port and config directory path via below command. Config directory is the directory where the `config.ini` and `key.json` files are located.
```shell
$ python3 main.py --host {host} --port {port} --config_dir {path to config directory}
```

## APIs
### Authorization
The operations which need authorization process like `upload`, `download` and `remove` should be requested with authorization headers as below. 

#### Request Headers
| Parameter | Type | Description |
| :--- | :--- | :--- |
| `X-Auth-Token` | `string` | **Required**. Received `ACCESS_TOKEN` |
| `X-Public-Key` | `string` | **Required**. User's public key (hex encoded) |
| `X-Signature` | `string` | **Required**. Signed `ACCESS_TOKEN` (hex encoded) |


To acquire `ACCESS_TOKEN`, client should have to send `POST` request to `auth` API with below data in the request body. 
```
{"user": user_identity, "operation": operation_description"}
```
The `user_indentity` is the user account address in [AMO ecosystem](https://github.com/amolabs/docs/blob/master/storage.md#user-identity) and `operation_description` is composed of operation's name and parcel id or hash data like described below.
```
/* for `download`, `remove` operations */
{"name": operation_name, "id": parcel_id"}

/* for `upload` operation only */
{"name": operation_name, "hash":data_body_256_hash"} 
```

The `operation_name` can be `"upload"`, `"download"`, `"remove"` and should be lowercase. 
`inspect` operation is not included because `inspect` operation should be requested without `auth`. For more detail, see the [Operation Description](https://github.com/amolabs/docs/blob/master/storage.md#api-operations). 

When the `ACCESS_TOKEN` is acquired, requesting client can construct request headers for an authorization. The request header must contains `X-Auth-Token`, `X-Public-Key`, `X-Signature` values. `X-Auth-Token` should be `ACCESS_TOKEN` value and `X-Public-Key` should be user's public key and it must be hex encoded format. Finally, `X-Signature` should be signed `ACCESS_TOKEN` which is signed with user's private key and it must be hex encoded format. Then, a client should send a request with those headers included.

#### Error 
When error is occurred, server will return the proper HTTP error code with response body : `{"error":{ERROR_MESSAGE}}`.

#### Body Parameter Detail
* Each request body is `JSON-encoded` format.
* Each API's request body parameter is well defined on [AMO storage Documents](https://github.com/amolabs/docs/blob/master/storage.md).

### Auth API
```http
POST /api/{api_version}/auth
```
#### Request Body

| Parameter | Type | Description |
| :--- | :--- | :--- |
| `user` | `string` | **Required**. user_identity |
| `operation` | `JSON object` | **Required**. operation_description |

#### Response Body

```
{
  "token": ACCESS_TOKEN
}
```
#### Errors
| Status Code | Error Message | Description |
| :--- | :--- | :--- |
| 400 | Reason why request body is invalid | Invalid request body |
| 405 | None | Invalid request method |

### Upload API
**Auth Required**
```http
POST /api/{api_version}/parcels
```
#### Request Body

| Parameter | Type | Description |
| :--- | :--- | :--- |
| `owner` | `string` | **Required**. user_identity |
| `metadata` | `JSON object` | **Required**. metadata |
| `data` | `string` | **Required**. hex_encoded_binary_sequence |

##### Metadata form
Metadata field is a schemeless JSON form, but the `owner` field must be included.
```
{
  "owner": user_identity, // Mandantory field
  ...
}
```

#### Response Body
```
{
  "id": data_parcel_id
}
```

#### Errors
| Status Code | Error Message | Description |
| :--- | :--- | :--- |
| 400 | Reason why request body is invalid | Invalid request body |
| 401 | One or more required fields do not exist in the header | |
| 401 | Invalid token | |
| 401 | Token does not exist | |
| 401 | Token does not have permission to perform the operation | |
| 401 | Verification failed | |
| 405 | None | Invalid request method |
| 409 | Parcel ID `parcel_id` already exists | |
| 500 | Error occurred on saving ownership and metadata | Database error |
| 500 | Ceph error message | |



### Download API
**Auth Required**
```http
GET /api/{api_version}/parcels/{parcel_id}
```
#### Request Body
| Parameter | Type | Description |
| :--- | :--- | :--- |

#### Response Body
```
{
  "id": data_parcel_id,
  "owner": user_indentity,
  "data": hex_encoded_binary_sequence
}
```
#### Errors
| Status Code | Error Message | Description |
| :--- | :--- | :--- |
| 400 | Reason why request body is invalid | Invalid request body |
| 401 | One or more required fields do not exist in the header | |
| 401 | Invalid token | |
| 401 | Token does not exist | |
| 401 | Token is only available to perform `operation_name` | |
| 401 | Verification failed | |
| 401 | No permission to download data parcel | |
| 405 | None | Invalid request method |
| 500 | Ceph error message | |


### Inspect API
```http
GET /api/{api_version}/parcels/{parcel_id}?key=metadata
```
#### Request Body
| Parameter | Type | Description |
| :--- | :--- | :--- |

#### Response Body
```
{
  "id": data_parcel_id,
  "owner": user_indentity,
  "metadata": metadata
}
```
#### Errors
| Status Code | Error Message | Description |
| :--- | :--- | :--- |
| 404 | None | parcel_id does not exist |
| 405 | None | Invalid request method |

### Remove API
**Auth Required**
```http
DELETE /api/{api_version}/parcels/{parcel_id}
```
#### Request Body
| Parameter | Type | Description |
| :--- | :--- | :--- |

#### Response Body
```
{}
```
#### Errors
| Status Code | Error Message | Description |
| :--- | :--- | :--- |
| 401 | One or more required fields do not exist in the header | |
| 401 | Invalid token | |
| 401 | Token does not exist | |
| 401 | Token does not have permission to perform the operation | |
| 401 | Verification failed | |
| 405 | Not allowed to remove parcel | |
| 410 | Parcel does not exist | |
| 500 | Error occurred on deleting ownership and metadata | Database error |
| 500 | Ceph error message | |
