# Airflow Installation

## Docker
Easy way to install docker
```
#uninstall all possible docker version before new installment
sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine

sudo yum install -y docker
```

Validation
```
sudo service docker start
sudo docker run hello-world
```
