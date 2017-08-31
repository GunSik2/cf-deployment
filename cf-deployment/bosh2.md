# CF deployment based on bosh2

## Deploy Bosh2
- Install bosh
```
wget https://s3.amazonaws.com/bosh-cli-artifacts/bosh-cli-2.0.28-linux-amd64
sudo mv bosh-cli-2.0.28-linux-amd64 /usr/local/bin/bosh
sudo chmod a+x /usr/local/bin/bosh
```

- Deploy the Director
```
- Create directory to keep 
$ mkdir bosh-1 && cd bosh-1

# Clone Director templates
$ git clone https://github.com/cloudfoundry/bosh-deployment


# Fill below variables (replace example values) and deploy the Director
$ export ACCESS_KEY_ID=AKI..
$ export SECRET_ACCESS_KEY=BVb.. 
$ bosh create-env bosh-deployment/bosh.yml \
    --state=state.json \
    --vars-store=creds.yml \
    -o bosh-deployment/aws/cpi.yml \
    -v director_name=bosh-1 \
    -v internal_cidr=172.31.16.0/20 \
    -v internal_gw=172.31.16.1 \
    -v internal_ip=172.31.16.6 \
    -v access_key_id=$ACCESS_KEY_ID \
    -v secret_access_key=$SECRET_ACCESS_KEY \
    -v region=us-east-1 \
    -v az=us-east-1b \
    -v default_key_name=gunsik-virginia \
    -v default_security_groups=[gunsik-virginia] \
    --var-file private_key=~/.ssh/gunsik-virginia.pem \
    -v subnet_id=subnet-78425221
```
- Connect to the Director.
```
# Configure local alias
$ bosh alias-env bosh-1 -e 172.31.16.6 --ca-cert <(bosh int ./creds.yml --path /director_ssl/ca)

# Log in to the Director
$ export BOSH_CLIENT=admin
$ export BOSH_CLIENT_SECRET=`bosh int ./creds.yml --path /admin_password`
$ bosh -e bosh-1 login

# Query the Director for more info
$ bosh -e bosh-1 env
```

## Deploy Full CF
- Update cloud config
``` 
$ bosh -e bosh-1 update-cloud-config cf-deployment/aws/cloud-config.yml \
  -v az1=us-east-1a -v internal_cidr_z1=172.31.0.0/20  -v internal_gw_z1=172.31.0.1  -v subnet_id_z1=subnet-4f26d839 \
  -v az2=us-east-1b -v internal_cidr_z2=172.31.16.0/20 -v internal_gw_z2=172.31.16.1  -v subnet_id_z2=subnet-78425221 \
  -v az3=us-east-1d -v internal_cidr_z3=172.31.48.0/20 -v internal_gw_z3=172.31.48.1  -v subnet_id_z3=subnet-203a390b    
```
- Deploy CF
```
$ bosh -e bosh-1 upload-stemcell https://bosh.io/d/stemcells/bosh-aws-xen-hvm-ubuntu-trusty-go_agent?v=3445.2

## Deploy on Multiple AZ
$ export SYSTEM_DOMAIN=lite.paasxpert.com
$ bosh -e bosh-1 -d cf deploy cf-deployment/cf-deployment.yml \
  --vars-store cf-vars.yml \
  -v system_domain=$SYSTEM_DOMAIN

## Deploy on Single AZ
$ bosh -e bosh-1 -d cf deploy cf-deployment/cf-deployment.yml \
  --vars-store cf-vars.yml \
  -v system_domain=$SYSTEM_DOMAIN \
  -o cf-deployment/aws/single-haproxy.yml \
  -o cf-deployment/operations/scale-to-one-az.yml
```

- Download manifest
```
bosh -e bosh-1 -d cf man > cf-z1.yml
```

## Test
- Install CF cli
```
wget -q -O - https://packages.cloudfoundry.org/debian/cli.cloudfoundry.org.key | sudo apt-key add -
echo "deb http://packages.cloudfoundry.org/debian stable main" | sudo tee /etc/apt/sources.list.d/cloudfoundry-cli.list
# ...then, update your local package index, then finally install the cf CLI
sudo apt-get update
sudo apt-get install cf-cli
```
- Install Golang
```
wget https://storage.googleapis.com/golang/go1.8.1.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.8.1.linux-amd64.tar.gz
echo 'export GOROOT=/usr/local/go' >> ~/.bash_profile
echo 'export PATH=$PATH:$GOROOT/bin' >> ~/.bash_profile
source ~/.bash_profile
```

- App push Test
```
./test_base_basic.sh
```
- Smoke Test
```
./test_base_smoke.sh
```

## Deploy Status
- Z3 deployment
```
$ bosh -e bosh-1 vms
Instance                                          Process State  AZ  IPs           VM CID               VM Type
api/1a7cdae8-5053-4aa1-ab75-750b7c9b8a07          running        z1  172.31.0.12   i-02a4f1d3d8d5c56a6  m3.large
api/d67a68f5-6cfd-4cb8-9d00-c45f84330ce6          running        z2  172.31.16.11  i-0e864265ffbd3662a  m3.large
blobstore/c4598fd2-76ad-482a-827f-4a2032c1a01f    running        z1  172.31.0.11   i-0840521ee06d6c13a  m3.large
cc-bridge/4c3e49ca-c821-456b-96a2-c5fa22ce29b2    running        z2  172.31.16.17  i-0068b4196b1b128b3  m3.medium
cc-bridge/b44b1cbf-edaa-46cb-acfa-1f643f824b06    running        z1  172.31.0.18   i-043de809ee023ac58  m3.medium
cc-clock/0c3dfc86-4423-48c9-ab79-4a36e9f84bf5     running        z2  172.31.16.16  i-087bc028117813df6  m3.large
cc-clock/7fb11a39-5986-4312-9563-9cba8683c005     running        z1  172.31.0.17   i-020dc686b41c5cb1f  m3.large
cc-worker/789f5bc9-6058-4add-bc18-fe9d9dbae632    running        z2  172.31.16.12  i-0764af53a2cf6c923  m3.medium
cc-worker/8019f098-206d-4c2e-b6b4-6df8952b4d13    running        z1  172.31.0.13   i-0796ccac09a9e9857  m3.medium
consul/40a3d5f0-3f09-47e9-acff-a59171d9f6c8       running        z1  172.31.0.4    i-013aea328ac0502fb  m3.medium
consul/74c5cb04-7ec0-4d19-84c9-aefbc9c419ad       running        z3  172.31.48.4   i-0bdaf1f188faa811c  m3.medium
consul/a8cae83d-438c-44ce-9651-a77368368653       running        z2  172.31.16.4   i-09f878140185e222f  m3.medium
diego-api/3c4c2a27-7273-4275-905d-36a2a469a137    running        z2  172.31.16.9   i-0c26e0baf0d50ed0e  m3.large
diego-api/e6ae374f-c605-4c34-a78a-4c9ed04e6a03    running        z1  172.31.0.9    i-00a005f5d99d15676  m3.large
diego-brain/2051b5b9-1cb4-4eec-9c36-ffce2f4ba0d3  running        z1  172.31.0.15   i-0f9e9949a65f4d281  m3.medium
diego-brain/e804fb7d-9c23-4399-b3c4-7db81f58994c  running        z2  172.31.16.14  i-00efbee8ef4260682  m3.medium
diego-cell/2e48a921-8fa3-44b8-b85f-fe28ae83f997   running        z2  172.31.16.15  i-0a57b847d56cbf890  r3.xlarge
diego-cell/5c1e801a-00e6-4ee1-9318-62a1e22198bf   running        z1  172.31.0.16   i-0cfd8600a1facd5fb  r3.xlarge
doppler/59a7abd2-c9a9-4c9b-b01f-b785aa1fdff2      running        z2  172.31.16.8   i-0aa5ad558ef3b6d4b  m3.medium
doppler/ab101e68-be91-4234-be09-f9878ef4fee6      running        z1  172.31.0.7    i-077e6af3b54fcbac7  m3.medium
etcd/6b694036-26e7-4016-baa3-c57a96254d08         running        z1  172.31.0.6    i-00c7ccff28a8ad700  m3.medium
etcd/a06ae21b-4117-4b99-bc9b-8540fc738157         running        z3  172.31.48.5   i-09220076fddf83eaf  m3.medium
etcd/a0ebcf14-98a6-4ace-b5e9-95122bc5706c         running        z2  172.31.16.7   i-08feb98c0b9de7581  m3.medium
haproxy/50d9de69-7e8b-4e33-8f2c-8627cfa126d1      failing        z1  172.31.0.20   i-08fb331470025cab6  m3.medium
haproxy/597bd2c5-634b-4558-b26c-a03828c35f2f      running        z2  172.31.16.19  i-0c56f7a6546fbcfbb  m3.medium
log-api/0b4578be-ce03-4eb6-9572-b6622121c4a8      running        z2  172.31.16.18  i-0dc599d0015806f33  t2.small
log-api/1728cd1e-6430-4f5b-828b-5e13d6b18acb      running        z1  172.31.0.19   i-0416ab2cecc544053  t2.small
mysql/e03c69ce-3fad-4e60-a86c-0d115502f151        running        z1  172.31.0.8    i-0f7090522f8659e3e  m3.large
nats/657deb4b-92c1-433e-98eb-736497109163         running        z1  172.31.0.5    i-0402e8caf6b33e993  c3.large
nats/f02ef8d8-e12e-4ea0-afa8-36f38cf56cb8         running        z2  172.31.16.5   i-0e9da81dc9e7fcc78  c3.large
router/08f22417-41bf-4ca0-891b-908599b30280       running        z1  172.31.0.14   i-0b0faa90e4f0322de  m3.medium
router/5645b2cc-64dc-4b50-9bdd-d257eb8c4ab8       running        z2  172.31.16.13  i-0209e401808515561  m3.medium
uaa/09eede7d-40d2-43c5-b54a-1a673080a57b          running        z2  172.31.16.10  i-06c2e30b1bb4abe84  m3.medium
uaa/34362a3b-442f-4b51-8d77-82360355608c          running        z1  172.31.0.10   i-025481e4fbb60b7cf  m3.medium
```
- Single Zone
```
$ bosh -e bosh-1 vms

Instance                                          Process State  AZ  IPs          VM CID               VM Type
api/1a7cdae8-5053-4aa1-ab75-750b7c9b8a07          running        z1  172.31.0.12  i-02a4f1d3d8d5c56a6  m3.large
blobstore/c4598fd2-76ad-482a-827f-4a2032c1a01f    running        z1  172.31.0.11  i-0840521ee06d6c13a  m3.large
cc-bridge/b44b1cbf-edaa-46cb-acfa-1f643f824b06    running        z1  172.31.0.18  i-043de809ee023ac58  m3.medium
cc-clock/7fb11a39-5986-4312-9563-9cba8683c005     running        z1  172.31.0.17  i-020dc686b41c5cb1f  m3.large
cc-worker/8019f098-206d-4c2e-b6b4-6df8952b4d13    running        z1  172.31.0.13  i-0796ccac09a9e9857  m3.medium
consul/40a3d5f0-3f09-47e9-acff-a59171d9f6c8       running        z1  172.31.0.4   i-013aea328ac0502fb  m3.medium
diego-api/e6ae374f-c605-4c34-a78a-4c9ed04e6a03    running        z1  172.31.0.9   i-00a005f5d99d15676  m3.large
diego-brain/2051b5b9-1cb4-4eec-9c36-ffce2f4ba0d3  running        z1  172.31.0.15  i-0f9e9949a65f4d281  m3.medium
diego-cell/5c1e801a-00e6-4ee1-9318-62a1e22198bf   running        z1  172.31.0.16  i-0cfd8600a1facd5fb  r3.xlarge
doppler/ab101e68-be91-4234-be09-f9878ef4fee6      running        z1  172.31.0.7   i-077e6af3b54fcbac7  m3.medium
etcd/6b694036-26e7-4016-baa3-c57a96254d08         running        z1  172.31.0.6   i-00c7ccff28a8ad700  m3.medium
haproxy/50d9de69-7e8b-4e33-8f2c-8627cfa126d1      running        z1  172.31.0.20  i-08fb331470025cab6  m3.medium
log-api/1728cd1e-6430-4f5b-828b-5e13d6b18acb      running        z1  172.31.0.19  i-0416ab2cecc544053  t2.small
mysql/e03c69ce-3fad-4e60-a86c-0d115502f151        running        z1  172.31.0.8   i-0f7090522f8659e3e  m3.large
nats/657deb4b-92c1-433e-98eb-736497109163         running        z1  172.31.0.5   i-0402e8caf6b33e993  c3.large
router/08f22417-41bf-4ca0-891b-908599b30280       running        z1  172.31.0.14  i-00180df4ea6d571d9  m3.medium
uaa/34362a3b-442f-4b51-8d77-82360355608c          running        z1  172.31.0.10  i-025481e4fbb60b7cf  m3.medium

17 vms

```
## Custom Config Files

- cf-deployment/aws/cloud-config.yml
```
azs:
- name: z1
  cloud_properties: {availability_zone: ((az1))}
- name: z2
  cloud_properties: {availability_zone: ((az2))}
- name: z3
  cloud_properties: {availability_zone: ((az3))}
  
compilation:
  workers: 3
  reuse_compilation_vms: true
  az: z1
  vm_type: large
  network: default

disk_types:
- disk_size: 1024
  name: 1GB
- disk_size: 5120
  name: 5GB
- disk_size: 10240
  name: 10GB
- disk_size: 100240
  name: 100GB


networks:
- name: default
  type: manual
  subnets:
  - range: ((internal_cidr_z1))
    gateway: ((internal_gw_z1))
    azs: [z1]
    reserved: [((internal_gw_z1))/30]
    dns: [8.8.8.8]
    cloud_properties: {subnet: ((subnet_id_z1))}
  - range: ((internal_cidr_z2))
    gateway: ((internal_gw_z2))
    azs: [z2]
    reserved: [((internal_gw_z2))/30]
    dns: [8.8.8.8]
    cloud_properties: {subnet: ((subnet_id_z2))}
  - range: ((internal_cidr_z3))
    gateway: ((internal_gw_z3))
    azs: [z3]
    reserved: [((internal_gw_z3))/30]
    dns: [8.8.8.8]
    cloud_properties: {subnet: ((subnet_id_z3))}
- name: vip
  type: vip

vm_types:
- name: xlarge
  cloud_properties: &xlarge {instance_type: t2.small, ephemeral_disk: {size: 30000, type: gp2}}
- name: large
  cloud_properties: &large {instance_type: t2.small, ephemeral_disk: {size: 30000, type: gp2}}
- name: medium
  cloud_properties: &medium {instance_type: t2.small, ephemeral_disk: {size: 10000, type: gp2}}
- name: small
  cloud_properties: &small {instance_type: t2.small, ephemeral_disk: {size: 10000, type: gp2}}
- name: c3.large
  cloud_properties: *large
- name: m3.large
  cloud_properties: *large
- name: m3.medium
  cloud_properties: *medium
- name: r3.xlarge
  cloud_properties: *xlarge
- name: t2.small
  cloud_properties: *small
- name: default
  cloud_properties: *small


vm_extensions:
- name: 5GB_ephemeral_disk
  cloud_properties: {ephemeral_disk: {size: 5000, type: gp2}}
- name: 10GB_ephemeral_disk
  cloud_properties: {ephemeral_disk: {size: 10000, type: gp2}}
- name: 50GB_ephemeral_disk
  cloud_properties: {ephemeral_disk: {size: 50000, type: gp2}}
- name: 100GB_ephemeral_disk
  cloud_properties: {ephemeral_disk: {size: 100000, type: gp2}}
- name: 500GB_ephemeral_disk
  cloud_properties: {ephemeral_disk: {size: 500000, type: gp2}}
- name: 1TB_ephemeral_disk
  cloud_properties: {ephemeral_disk: {size: 1000000, type: gp2}}
- name: ssh-proxy-and-router-lb
  cloud_properties:
    ports:
    - host: 80
    - host: 443
    - host: 2222
- name: cf-tcp-router-network-properties
  cloud_properties:
    ports:
    - host: 1024-1123
```

- cf-deployment/aws/single-haproxy.yml
```
- type: replace
  path: /releases/-
  value:
    name: haproxy
    version: 8.1.1
    url: https://bosh.io/d/github.com/cloudfoundry-incubator/haproxy-boshrelease?v=8.1.1
    sha1: 98ec4d0571b5fccb975eab3ed5e40842e35b8daa
- type: replace
  path: /instance_groups/name=smoke-tests
  value:
    name: haproxy
    azs: [z1]
    instances: 1
    stemcell: default
    vm_type: m3.medium
    networks: [ name: default ]
    jobs:
    - name: haproxy
      release: haproxy
      properties:
        ha_proxy:
          ssl_pem: "((router_ssl.certificate))((router_ssl.private_key))"
          tcp_link_port: 2222
- type: replace
  path: /instance_groups/-
  value:
    name: smoke-tests
    lifecycle: errand
    azs:
    - z1
    instances: 1
    vm_type: m3.medium
    stemcell: default
    update:
      max_in_flight: 1
      serial: true
    networks:
    - name: default
    jobs:
    - name: smoke_tests
      release: cf-smoke-tests
      properties:
        smoke_tests:
          api: "https://api.((system_domain))"
          apps_domain: "((system_domain))"
          user: admin
          password: "((cf_admin_password))"
          org: cf_smoke_tests_org
          space: cf_smoke_tests_space
          cf_dial_timeout_in_seconds: 300
          skip_ssl_validation: true
```

- test_base_basic.sh
```
#!/bin/bash


cf create-org org
cf create-space space -o org
cf target -o "org" -s "space"


cf enable-feature-flag diego_docker

mkdir -p /tmp/test-php; cd /tmp/test-php
echo "<?php phpinfo(); ?>" > index.php
cf push test-php -b php_buildpack -m 64m
cf apps
cd -

cf push docker -o cloudfoundry/test-app -m 64m
cf apps
```

- test_base_smoke.sh
```
#!/bin/bash

#set -o +x

user=admin
install_path=/tmp/go

function input() {
    echo -n "app domain = "
    read apps_domain
    echo -n "admin password = "
    read password

    echo $apps_domain
    echo $password
    target=api.$apps_domain
}

function install() {
    go get github.com/tools/godep
    godep save

    go get github.com/kr/godep
}


function config() {
    cat > smoke_test.json <<EOF
  {
      "suite_name"           : "CF_SMOKE_TESTS",
      "api"                  : "$target",
      "apps_domain"          : "$apps_domain",
      "user"                 : "$user",
      "password"             : "$password",
      "org"                  : "CF-SMOKE-ORG",
      "space"                : "CF-SMOKE-SPACE",
      "cleanup"              : true,
      "use_existing_org"     : false,
      "use_existing_space"   : false,
      "logging_app"          : "",
      "runtime_app"          : "",
      "enable_windows_tests" : true,
      "backend"              : "diego",
      "skip_ssl_validation"  : true,
      "enable_windows_tests" : false,
      "artifacts_directory"  : "/tmp/smoke-artifacts"
  }
EOF
}


function run() {
    export CONFIG=$(pwd)/smoke_test.json
    go get -d github.com/cloudfoundry/cf-smoke-tests
    cd $GOPATH/src/github.com/cloudfoundry/cf-smoke-tests/
    ./bin/test -v
}

function env() {
    mkdir -p $install_path; cd $install_path
    export GOPATH=$install_path
    export GOROOT=/usr/local/go
    export PATH=$PATH:$GOPATH/bin
}

function main() {
    input
    env
    install
    config
    run
}

main
```

- test_base_cat.sh
```
#!/bin/bash

#set -o -x

install_path=/tmp/cat
git_src_path=$install_path/src/github.com/cloudfoundry/cf-acceptance-tests
xpert_deploy=$(pwd)

function input() {
    echo -n "app domain = "
    read apps_domain
    echo -n "admin password = "
    read admin_password

    echo $apps_domain
    echo $password
    target=api.$apps_domain
}

function install() {
    cd $install_path
    go get -d github.com/cloudfoundry/cf-acceptance-tests

    cd $git_src_path
    ./bin/update_submodules
    go get -u github.com/FiloSottile/gvt
    gvt restore
}


function config() {
    cat > $install_path/cat_test.json <<EOF
  {
      "api"                     : "$target",
      "apps_domain"             : "$apps_domain",
      "admin_user"              : "admin",
      "admin_password"          : "$admin_password",
      "backend"                 : "diego",
      "skip_ssl_validation"     : true,
      "use_http"                : true,
      "include_apps"            : true,
      "include_backend_compatibility"   : false,
      "include_container_networking"    : true,
      "include_internet_dependent"      : true,
      "include_security_groups"         : true,
      "include_detect"                  : false,
      "include_docker"                  : true,
      "include_privileged_container_support": true,
      "include_route_services"  : false,
      "include_routing"         : true,
      "include_zipkin"          : true,
      "include_services"        : true,
      "include_ssh"             : true,
      "include_sso"             : false,
      "include_tasks"           : false,
      "include_v3"              : false,
      "artifacts_directory"     : "$install_path/cat-artifacts"
  }
EOF
}

function patch_core() {
    echo
}

function run() {
    export CONFIG=$install_path/cat_test.json

    cd $git_src_path
    ./bin/test -nodes=2
}

function package() {
    sudo add-apt-repository ppa:bzr/ppa
    sudo apt-get update
}

function env() {
    mkdir -p $install_path;
    export GOPATH=$install_path
    export GOROOT=/usr/local/go
    export PATH=$PATH:$GOPATH/bin
}
function patch_core() {
    echo
}

function run() {
    export CONFIG=$install_path/cat_test.json

    cd $git_src_path
    ./bin/test -nodes=2
}

function package() {
    sudo add-apt-repository ppa:bzr/ppa
    sudo apt-get update
}

function env() {
    mkdir -p $install_path;
    export GOPATH=$install_path
    export GOROOT=/usr/local/go
    export PATH=$PATH:$GOPATH/bin
}

function cleanup() {
    cf delete-org -f CF-SMOKE-ORG
    cf delete-quota -f CF-SMOKE-ORG_QUOTA
}

function main() {
    install
    config
    run
}

function help() {
    echo "PaaSXpert acceptance tests"
}

env
input
main
#$*
```

## Reference
- Bosh2 manifest : http://bosh.io/docs/manifest-v2.html
- Bosh2 cloud-config : https://bosh.io/docs/cloud-config.html
- Bosh2 cloud-config for aws : https://bosh.io/docs/aws-cpi.html#cloud-config
- Bosh2 cloud-config sample : https://github.com/pivotal-cf/p-ert-bosh-experiment/blob/master/aws/cloud-config.yml
- Bosh2 cli : https://bosh.io/docs/cli-v2.html
- Deploy bosh2 for aws : https://bosh.io/docs/init-aws.html
- Deploy cf : https://github.com/cloudfoundry/cf-deployment
