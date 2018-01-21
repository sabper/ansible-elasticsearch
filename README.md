# Elasticsearch

* single master / data node elasticsearch

* 3 master / data node elasticsearch cluster

* Add data node to elasticsearch cluster

* metricbeat install

<br>

## ansible 수행 전 필요 사항

### ec2 instance

* es 설치할 ec2 instance 시작

* es 설치할 ec2 instance 에 ssh 접속이 가능한지 확인

  * es cluster ec2 instance private subnet에 존재 - 동일한 vpc 내 ec2 instance 에서 ansible 실행

  * es cluster ec2 instance public subnet에 존재 - 로컬에서 ansible 실행

* es 설치하려는 instance로 ssh 접속 가능한 환경에서 inventory 설정

  ```shell
  $ cat /etc/ansible/hosts
  [Elasticsearch]
  {ec2 instance ip} ansible_ssh_private_key_file={pem file path}

  $ ansible Elasticsearch -m ping -u ec2-user
  172.31.35.237 | SUCCESS => {
      "changed": false,
      "ping": "pong"
  }
  ```

* ansible repo git clone

* metricbeat 가 바라보는 kibana는 이미 설치 완료되어 있어야 한다. 순서는 `es -> kibana -> metricbeat`
  * **kibana 정상 동작 중이 (es 연결 완료) 아니면 kibana 정상 동작 후에 각 metricbeat 재시작해줘야 함.**
  * **metricbeat kibana 정상 접속 후에는 kibana 상태와 상관없이 metricbeat 정상 동작 함. 재시작 필요 없음**

<br>

## ES 설치

* es 설치가 완료 후 kibana <-> ES cluster 접속 확인 후 es cluster 각 ec2 instance에 metricbeat 설치

* es 최초 설치 시에는 (single, cluster 모두) es 설치 완료 후 kibana <-> es 접속 확인 후 metricbeat 설치해야함
  * kibana <-> es 세팅 전 metricbeat 설치 시 metricbeat에서 kibana 확인하지 못하여 kibana 설정 완료 후 metricbeat 재구동 해야 함

### es single

* `site.yml` 수정

  * `network.host` ec2 instance private ip 로 세팅 `network.host: 172.31.14.112`

  * `discovery.zen.ping.unicast.hosts` ec2 instance private ip 로 세팅 `discovery.zen.ping.unicast.hosts: "172.31.14.112:1301"`

  * `hosts` `/etc/ansible/hosts` 에서 설정한 이름으로 세팅 `es_instance_name: "node-1"`

* es single node ansible-playbook 실행

  ```shell
  ansible-playbook site.yml
  ```

* **metricbeat 필요할 경우 metricbeat playbook 별도로 구성하여 실행 해야함**

<br>

### es cluster 최초 구성 - 3 node (data, master)

준비된 ec2 instance 에 es cluster 초기 구성 세팅

#### es cluster 구성

* `site.yml` 수정

  * `network.host` ec2 instance private ip 로 세팅 `network.host: 172.31.18.55`

  * `discovery.zen.ping.unicast.hosts` ec2 instance private ip 로 세팅

    ```code
    discovery.zen.ping.unicast.hosts: "172.31.1.186:1301,172.31.14.239:1302,172.31.18.55:1303",
    ```

  * `hosts` `/etc/ansible/hosts` 에서 설정한 이름으로 세팅

    ```code
    hosts: Elasticsearch-1
    hosts: Elasticsearch-2
    hosts: Elasticsearch-3
    ```

* es cluster ansible playbook 실행

  ```shell
  ansible-playbook es-cluster-three.yml
  ```

#### metricbeat 설치

* **설치된 es cluster 와 kibana 정상 접속 됐는지 확인 필요**

* `instance_name` ec2 instance host name

  ```code
  instance_name: "node-2"
  ```

* `metricbeat_hosts` es private ip

  ```code
  metricbeat_hosts: "172.31.1.186:1201\", \"172.31.14.239:1202\", \"172.31.18.55:1203"
  ```

* `kibana_host` kibana host ip

  ```code
  kibana_host: "172.31.8.104:5601"
  ```

* metricbeat ansible playbook 실행

  ```shell
  ansible-playbook metric-three.yml
  ```

<br>

### es cluster 에 데이터 노드 추가

* 존재하는 cluster 에 데이터 노드 추가 시나리오

* 기존 cluster node는 설정 변경이 필요 없음

* kibana와 es 가 붙어있는 상황이므로 metricbeat 동시 설치 가능

* `site.yml` 수정

  * `network.host` ec2 instance private ip 로 세팅 `network.host: 172.31.18.55`

  * `discovery.zen.ping.unicast.hosts` ec2 instance private ip 로 세팅

    ```code
    discovery.zen.ping.unicast.hosts: "172.31.1.186:1301,172.31.14.239:1302,172.31.18.55:1303",
    ```

  * `hosts` `/etc/ansible/hosts` 에서 설정한 이름으로 세팅

    ```code
    hosts: Elasticsearch-1
    hosts: Elasticsearch-2
    hosts: Elasticsearch-3
    ```

  * `metricbeat_hosts` es private ip

    ```code
    metricbeat_hosts: "172.31.1.186:1201\", \"172.31.14.239:1202\", \"172.31.18.55:1203"
    ```

  * `kibana_host` kibana host ip

    ```code
    kibana_host: "172.31.8.104:5601"
    ```

* es cluster에 data node 추가 ansible playbook 실행

  ```shell
  ansible-playbook add-data-node.yml
  ```
