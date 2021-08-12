# crypto_etl_dashboard


This is an ETL pipeline to pull bitcoin exchange data from [CoinCap API](https://docs.coincap.io/) and load it into a data warehouse. 


## 
**Architecture**


![alt_text](https://github.com/sanjaykmenon/crypto_etl_dashboard/blob/main/bc_arch.png "image_tooltip")


I’ve used python to pull, transform and load data. The warehouse is postgres. As a presentation layer I’ve also added a dashboard interface on Metabase for business stakeholders to view metrics.

All of the components are running as docker containers.


## 
**Setup**


### 
**Pre-requisites**



1. [Docker](https://docs.docker.com/engine/install/) and [Docker Compose](https://docs.docker.com/compose/install/) v1.27.0 or later.
2. [AWS account](https://aws.amazon.com/).
3. [AWS CLI installed](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html) and [configured](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html).
4. [git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git).

### 
**Local**


A Makefile is used for deploying on Docker with common commands. These are executed in the running container.


```
cd crypto_etl_dashboard

make up # starts all the containers
make ci # runs formatting, lint check, type check and python test
```


If the CI step passes you can go to [http://localhost:3000](http://localhost:3000/) to checkout your Metabase instance.

One can connect to the warehouse with the following credentials


```
Host: warehouse
Database name: finance
```


The remaining configs are available in the env file.

Refer to [this doc](https://www.metabase.com/docs/latest/users-guide/07-dashboards.html) for creating a Metabase dashboard.


### 
**Production**

Helper scripts are used to deploy containers to AWS EC2 deploy_helpers for this.

I used an `ubuntu x_86` EC2 instance with a custom TCP inbound rule with port `3000` open to the IP `0.0.0.0/0`. These can be set when you create an AWS EC2 instance in the `configure security group` section. A `t2.micro` (free-tier eligible) instance would be sufficient.


You can setup a prod instance as shown below.


```
cd crypto_etl_dashboard

chmod 755 ./deploy_helpers/send_code_to_prod.sh

chmod 400 pem-full-file-location

./deploy_helpers/send_code_to_prod.sh pem-full-file-location EC2-Public-IPv4-DNS

# install docker on your Ubuntu EC2 instance
chmod 755 install_docker.sh

./install_docker.sh
# verify that docker and docker compose installed
docker --version
docker-compose --version

# start the containers
unzip crypto_etl_dashboard.gzip && cd crypto_etl_dashboard/
docker-compose --env-file env up --build -d
```



## 
**Tear down**

You can spin down your local instance with.


```
make down
```


## **Discussion**


**ETL Code**

The code to pull data from [CoinCap API](https://docs.coincap.io/) and load it into the warehouse is at [exchange_data_etl.py](https://github.com/sanjaykmenon/crypto_etl_dashboard/blob/main/src/bitcoinmonitor/exchange_data_etl.py). 

In this script, we :



1. Pull data from [CoinCap API](https://docs.coincap.io/) using the get_exchange_data function.
2. Use get_utc_from_unix_time function to get UTC based date time from unix time(in ms).
3. Load data into our warehouse using the _get_exchange_insert_query insert query.


```
def run() -> None:
    data = get_exchange_data()
    for d in data:
        d['update_dt'] = get_utc_from_unix_time(d.get('updated'))
    with WarehouseConnection(**get_warehouse_creds()).managed_cursor() as curr:
        p.execute_batch(curr, _get_exchange_insert_query(), data)
```


**Tests**

Two kinds of tests are performed: 



1. **Unit test**: To test if individual functions are working as expected. We test get_utc_from_unix_time with the test_get_utc_from_unix_time function.
2. **Integration test**: To test if multiple systems work together as expected.

For the integration test we

1. Mock the Coinbase API call using the mocker functionality of the pytest-mock library. We use fixture data at test/fixtures/sample_raw_exchange_data.csv as a result of an API call. This is to enable deterministic testing.
2. Assert that the data we store in the warehouse is the same as we expected.


**Further things to do**

Lots of future work still left on this but a few that come to mind:

1. CI/CD with Github actions.
2. Data quality testing using dbt?
3. Better scheduler and workflow manager to handle backfills, reruns, and parallelism.
4. Streaming data from APIs vs mini-batches.
5. Add system env variable to crontab.
6. Data cleanup job to remove old data, since our Postgres is running on a small EC2 instance.
7. API rate limiting.


