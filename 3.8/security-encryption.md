---
layout: default
description: Securing physical storage media, available in the Enterprise Edition
title: Encryption at Rest
---
# Encryption at Rest

{% include hint-ee-oasis.md feature="Encryption at rest" %}

When you store sensitive data in your ArangoDB database, you want to protect
that data under all circumstances. At runtime you will protect it with SSL
transport encryption and strong authentication, but when the data is already
on disk, you also need protection. That is where the Encryption feature comes
in.

The Encryption feature of ArangoDB will encrypt all data that ArangoDB is
storing in your database before it is written to disk.

The data is encrypted with AES-256-CTR, which is a strong encryption algorithm,
that is very suitable for multi-processor environments. This means that your
data is safe, but your database is still fast, even under load.

Most modern CPU's have builtin support for hardware AES encryption, which makes
it even faster.

The encryption feature is supported by all ArangoDB deployment modes.

## Limitations

The encryption feature has the following limitations:

- Encrypting a single collection is not supported: all the databases are
  encrypted.
- It is not possible to enable encryption at runtime: if you have existing
  data you will need to take a backup first, then enable encryption and
  start your server on an empty data-directory, and finally restore your
  backup.  
- The Encryption feature requires the RocksDB storage engine.

## Encryption keys

The encryption feature of ArangoDB requires a single 32-byte key per server.
It is recommended to use a different key for each server (when operating in a
cluster configuration).

{% hint 'security' %}
Make sure to protect the encryption keys! That means:

- Do not write them to persistent disks or your server(s), always store them on
  an in-memory (`tmpfs`) filesystem.

- Transport your keys safely to your server(s). There are various tools for
  managing secrets like this (e.g.
  [vaultproject.io](https://www.vaultproject.io/){:target="_blank"}).

- Store a copy of your key offline in a safe place. If you lose your key, there
  is NO way to get your data back.
{% endhint %}

## Configuration

To activate encryption of your database, you need to supply an encryption key
to the server.

Make sure to pass this option the very first time you start your database.
You cannot encrypt a database that already exists.

Note: You also have to activate the RocksDB storage engine.

### Encryption key stored in file

Pass the following option to `arangod`:

```
$ arangod \
    --rocksdb.encryption-keyfile=/mytmpfs/mySecretKey \
    --server.storage-engine=rocksdb
```
The file `/mytmpfs/mySecretKey` must contain the encryption key. This
file must be secured, so that only `arangod` can access it. You should
also ensure that in case someone steals the hardware, he will not be
able to read the file. For example, by encrypting `/mytmpfs` or
creating an in-memory file-system under `/mytmpfs`.

### Encryption key generated by a program

Pass the following option to `arangod`:

```
$ arangod \
    --rocksdb.encryption-key-generator=path-to-my-generator \
    --server.storage-engine=rocksdb
```

The program `path-to-my-generator` output the encryption on standard
output and exit.

### Kubernetes encryption secret

If you use _kube-arangodb_ then use the `spec.rocksdb.encryption.keySecretName`
setting to specify the name of the Kubernetes secret to be used for encryption.
See [Kubernetes Deployment Resource](deployment-kubernetes-deployment-resource.html#specrocksdbencryptionkeysecretname).

## Creating keys

The encryption keyfile must contain 32 bytes of random data.

You can create it with a command line this.

```
dd if=/dev/random bs=1 count=32 of=yourSecretKeyFile
```

For security, it is best to create these keys offline (away from your database
servers) and directly store them in your secret management tool.

## Rotating encryption keys

ArangoDB supports rotating the user supplied encryption at rest key.
This is implemented via key indirection. The user supplied key is used 
to encrypt a randomly generated internal master key.

It is possible to change the user supplied encryption at rest key via the
[HTTP API](http/administration-and-monitoring.html#encryption-at-rest).

To enable smooth rollout of new keys you can use the new option 
`--rocksdb.encryption-keyfolder` to provide a set of secrets.
_arangod_ will then store the master key encrypted with the provided secrets.

```
$ arangod \
    --rocksdb.encryption-keyfolder=/mytmpfs/mySecrets
```

To start an arangod instance only one of the secrets needs to be correct, 
this should guard against service interruptions during the rotation process.