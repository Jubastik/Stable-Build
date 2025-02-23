# Регистрация gitlab-runner

``` bash
sudo gitlab-runner register -n \
  --url "http://192.168.31.130" \
  --registration-token GR1348941vyLt81yLK8PhPYgyTu6r \
  --executor docker \
  --description "My Docker Runner" \
  --docker-image "docker:27.5.1" \
  --docker-privileged
  ```