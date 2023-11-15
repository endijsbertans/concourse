# Concourse Docker
Please clone this repo
```sh
git clone https://github.com/endijsbertans/concourse.git
```
Then change localhost to YOUR public IPV4
```sh
cd concourse
nano docker-compose.yml
```
![image](https://github.com/endijsbertans/concourse/assets/97877531/26d57c17-7b03-4282-bf64-d1514f2e5345)

you'll first need to execute
`./keys/generate` - this will generate credentials used to authorize the
Concourse components with each-other:

```sh
$ ./keys/generate
wrote private key to /keys/session_signing_key
wrote private key to /keys/tsa_host_key
wrote ssh public key to /keys/tsa_host_key.pub
wrote private key to /keys/worker_key
wrote ssh public key to /keys/worker_key.pub
```

Next, run `docker-compose up -d` to start Concourse in the background:

```sh
$ docker-compose up -d
Starting concourse-docker_db_1 ... done
Starting concourse-docker_web_1 ... done
Starting concourse-docker_worker_1 ... done
```

The default configuration sets up a `test` user with `test` as their password
and grants them access to `main` team. To use this in production you'll
definitely want to change that - see [Auth &
Teams](https://concourse-ci.org/auth.html) for more information..

If things seem to be going wrong, check the logs for any errors:

```sh
$ docker-compose logs -f
Attaching to concourse-docker_worker_1, concourse-docker_web_1, concourse-docker_db_1
...
```
WHOLAA! check if it woorks!
![image](https://github.com/endijsbertans/concourse/assets/97877531/f37233d5-06ad-46c7-a2dc-b84ead8babf0)

## Building `concourse/concourse`

The `Dockerfile` in this repo is built as part of our CI process - as such, it
depends on having a pre-built `linux-rc` available in the working directory, and
ends up being published as `concourse/concourse`.
