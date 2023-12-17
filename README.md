# bookwyrm-kubernetes

This is a collection of files that I used to run [Bookwyrm](https://joinbookwyrm.com) on Kubernetes.

Bookwyrm originally runs with Docker-compose, which is not super-scalable. The installation instruction is made with the assumption that some infrasctructure, such as redis and database, isn't already available.

## My installation

The configuration files on this repo assume that I am:

- using only one redis instance for both activity and broker,
- using S3 for storage
- having an external reverse-proxy in addition to my nginx-ingress on my k8s cluster.
- using an already configured reverse proxy
- relying on an already present postgres install
- using a https proxy for the S3 files, so that all files are shown from my domain, and not directly from my S3 provider. I prefer this so that it is easier to migrate to another bucket later. A good example on how to do this is on Mastodon documentation: https://docs.joinmastodon.org/admin/optional/object-storage-proxy/

## Installation procedure

- Clone Bookwyrm-repo:  
`git clone git@github.com:bookwyrm-social/bookwyrm.git`
- Checkout to the `production` branch: 
`git checkout production`
- build the image (You need to build the image and upload it to your container registry):
`docker buildx build --platform=linux/amd64 -t mycontainerregistry/bookwyrm:0.6.6 . --no-cache`
`docker push mycontainerregistry/bookwyrm:0.6.6`
- Configure your persistant volume (I used nfs) on k8s with the following folders:
 - redis
 - static
 - images
 - app
- Configure your nginx file, according to https://docs.joinbookwyrm.com/reverse-proxy.html. Usually, you copy `/app/nginx/production` to `default.conf`, and edit accordingly.
- Copy redis.conf to the root of your persistant volume
- Configure your `configMap.yml` and `secrets.yml`, as well as all the deployments, so that they point to your volume and container registry,
- After everything is configured, deploy these:
```
kubectl apply -f configMap.yml
kubectl apply -f secrets.yml
kubectl apply -f bookwyrm-service*
kubectl apply -f bookwyrm-ingress.yml
kubectl apply -f bookwyrm-worker.yml
```
- Get a shell on your worker pod, and execute the following commands:
```
python manage.py migrate
python manage.py migrate django_celery_beat
python manage.py initdb
```
- Now get the other deployments up:
```
kubectl apply -f bookwyrm-beat.yml
kubectl apply -f bookwyrm-flower.yml
kubectl apply -f bookwyrm-web.yml
```
- Finally, run the following commands on the web pod:
```
python manage.py compile_themes
python manage.py collectstatic --no-input
python manage.py admin_code
```

## Considerations

I am not sure how easy it will be to update this on kubernetes, and I'm surely duplicating a lot of unnecessary work here. I believe things will be easier if I modify the image to actually copy the installation instead of mounting a volume for it, but I want to confirm this with the developers to make sure these files are purely static.

I had a bit of trouble with my instance as it was not respecting the `USE_HTTPS: "true"` configuration, so I created an `.env`  file on my `/app` folder and added `USE_HTTPS=true`. It worked well after that.

It is most likely easier to maintain a Helm chart for this, but I'm not so familiar with Helm charts, so I decided to go for this simple setup.

The cool thing is that if my instance needs to grow, all I need is to increase the number of replicas for the woker nodes and for the web nodes.


