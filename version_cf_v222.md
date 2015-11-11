* BOSH Release Version: [bosh/201](https://bosh.io/releases/github.com/cloudfoundry/bosh?version=201)
* BOSH Stemcell Version(s): [bosh-aws-xen-kvm-ubuntu-trusty-go_agent/3104](http://boshartifacts.cloudfoundry.org/file_collections?type=stemcells)
* diego: v0.1437.0   (bosh direcotr: 1.2859.0)
* Garden-linux:  [v0.308.0](https://bosh.io/releases/github.com/cloudfoundry-incubator/garden-linux-release?version=0.308.0) 
* Etcd final release [16](http://bosh.io/releases/github.com/cloudfoundry-incubator/etcd-release?version=16)

* bosh-init config
```
releases:
- name: bosh
  url: https://bosh.io/d/github.com/cloudfoundry/bosh?v=201
  sha1: 2264a48ffbca0fba70a19c4ae8e73f510839bac1
- name: bosh-openstack-cpi
  url: https://bosh.io/d/github.com/cloudfoundry-incubator/bosh-openstack-cpi-release?v=16
  sha1: d7ef4b16db1312a134b2a18de6ac4bea520fafa9

resource_pools:
- name: vms
  network: private
  stemcell:
    url: https://bosh.io/d/stemcells/bosh-openstack-kvm-ubuntu-trusty-go_agent?v=3104
    sha1: 0e0f48f064663b13990c77092e265ae185db1db2
  cloud_properties:
    instance_type: $INSTANCE_TYPE    # <-- Replace / Added

```


* Reference
  - https://bosh.cloudfoundry.org/releases/github.com/cloudfoundry/cf-release?version=222
  - https://github.com/cloudfoundry-incubator/diego-cf-compatibility/blob/master/compatibility-v1.csv
  - https://bosh.io/releases/github.com/cloudfoundry-incubator/bosh-openstack-cpi-release?version=16
  - https://bosh.io/docs/init-openstack.html
  - http://docs.cloudfoundry.org/deploying/openstack/
