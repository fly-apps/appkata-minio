# Appkata-minio

Deploy your own S3 compatible storage with MinIO and Fly

<!---- cut here --->

## Rationale

**Appkata**: The application fighting style - and a series of application-centric examples for Fly.io.

As you build out your applications, the need for storage of larger unstructured data, such as web assets, images, or any kind of large file, may become essential. Software compatible with Amazon's S3 is one of the ways people address this issue - the other, obviously being using third party services such as Amazon's S3 to store data on. [MinIO](https://min.io/) is an object storage system available under the AGPLv3 which can provide such an S3 compatible service.

We're going to set up MinIO on Fly, backed by Fly's disk Volume storage, so we can use it as part of a digital asset delivery system.

## Preparing

For this app, we'll use the official [minio/minio](https://hub.docker.com/r/minio/minio) Docker image. We'll use a Dockerfile, a very minimal Dockerfile:

```docker
FROM minio/minio

CMD [ "server", "/data" ]
```

This just takes the MinIO image and runs it with the command `server /data`, so that the server runs using the /data directory. We'll get to that directory in a moment.

## Initializing
This image uses port 9000 to communicate with the world. We have a Dockerfile. We're now ready to initialize our Fly app. Using `fly init` we create an app called `appkata-minio`. We then add the `--dockerfile` flag to tell it to use the Dockerfile we created, `--org personal` to deploy it to our personal organization, and `--port 9000` to set the default internal port to 9000.

```cmd
fly init appkata-minio --dockerfile --org personal --port 9000
```
```out
Selected App Name: appkata-minio

New app created
  Name         = appkata-minio
  Organization = personal
  Version      = 0
  Status       =
  Hostname     = <empty>

App will initially deploy to lhr (London, United Kingdom) region

Wrote config file fly.toml
```

The generated configuration in fly.toml will now send connections over HTTP (port 80) and HTTPS (port 443) to port 9000 of the running image. Make a note of where the app will initially deploy, we'll need that in a moment.

## Disk Storage

The application's VM and image are ephemeral. When the app is stopped of moved it loses any data written to its file system. That's why there are Volumes, storage that is independent of the VM and image. All we need to do is create a Volume with a name and a size in the region where the app is. 

```cmd
❯ fly vol create miniodata --region lhr
```
```out
        ID: L5365X5p7QRzxfv6eJZ8
      Name: miniodata
    Region: lhr
   Size GB: 10
Created at: 09 Nov 20 14:34 UTC
```

If not specified, the size defaults to the minimum: 10GB. Now we have that volume, we need to let out app know about it. Back to the `fly.toml` and append to the file:

```toml
[mounts]
source="miniodata"
destination="/data"
```

This mounts the volume we just created onto the '/data' directory of the running app. With this in place, it's now time to set some secrets.

## Secrets

MinIO uses two environment variables that can be set to provide an access key and secret key which are used to log into the server as admin. To pass these values to the server, we'll use `fly secrets set` which will encrypt the values being passed and only decrypt them at runtime, using the key names to set similarly named environment variables.

```cmd
fly secrets set MINIO_ACCESS_KEY=appkataaccesskey MINIO_SECRET_KEY=appkatasecretkey
```

## Deploying

With our configuration complete, we can now deploy the app:

```cmd
fly deploy
```

When the deployment is complete, run `fly open` to browse to the server. MinIO has a web interface for creating buckets and opening files. You'll need to log in first with the access key and secret key you saved into secrets.

If you have the [MinIO Client](https://docs.min.io/docs/minio-client-quickstart-guide.html) installed, you can also set this up as an alias to connect to with:

```cmd
mc alias set appkata https://appkata-minio.fly.dev/ appkataaccesskey appkatasecretkey
```

And proceed to administer the server. 


## Discuss

* You can discuss this example on its dedicated [Fly Community](https://community.fly.io/t/appkata-minio-s3-compatible-storage-with-fly/389) topic.
