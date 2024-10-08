# IoT Device Management API

## Table of Contents

- [Getting Started](https://github.com/keshav1002/iot-device-management-api?tab=readme-ov-file#getting-started)
- [Scenario: Smart Building Temperature Monitoring System](https://github.com/keshav1002/iot-device-management-api?tab=readme-ov-file#scenario-smart-building-temperature-monitoring-system)
- [API Documentation](https://github.com/keshav1002/iot-device-management-api?tab=readme-ov-file#api-documentation)
- [High-Level Architecture and Database Design](https://github.com/keshav1002/iot-device-management-api?tab=readme-ov-file#high-level-architecture-and-database-design)
- [Unit test](https://github.com/keshav1002/iot-device-management-api?tab=readme-ov-file#unit-test)
- [Integration test](https://github.com/keshav1002/iot-device-management-api?tab=readme-ov-file#integration-test)
- [CI/CD](https://github.com/keshav1002/iot-device-management-api?tab=readme-ov-file#cicd)
- [Sonarcloud](https://github.com/keshav1002/iot-device-management-api?tab=readme-ov-file#sonarcloud)
- [Logs](https://github.com/keshav1002/iot-device-management-api?tab=readme-ov-file#logs)

## Getting Started

The API was built using the [Serverless Framework](https://www.serverless.com/) v4, NodeJS v20 and Typescript v5 and can be deployed to the AWS cloud. To get started clone the repo and make sure the following prerequisites are installed in your machine.

### Prerequisites

1. [NodeJS](https://nodejs.org/) v20 or above.
2. AWS Account along with the [AWS CLI v2](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) configured.
3. [Docker Desktop](https://www.docker.com/products/docker-desktop/) installed and configured along with [Docker compose](https://docs.docker.com/compose/install/). (This is needed for [DynamoDB Local](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DynamoDBLocal.DownloadingAndRunning.html#docker))
4. Optional - [Prettier](https://prettier.io/docs/en/editors) installed in your IDE for code formatting. (Preferably [VS Code](https://code.visualstudio.com/download))

### Setting up Local Development

After cloning the repo, run the following command to install all the required dependencies.

```
npm install
```

The project is configured to connect to DynamoDB Local when doing local development. DynamoDB local can be setup locally in different ways. For convenience and ease of use a docker compose file can be found under `scripts/docker-compose.yml`. Run the following command inside that directory to start the local docker instance.

```
docker compose up
```

A convenient script can also be found under the same folder `scripts/create-table.mjs`. Run this script using the following command to create the necessary tables locally for the API. Make sure to run this script after the DynamoDB Docker instance is up and running.

```
node create-table.mjs
```

Finally you can refer to the `.env.sample` file to configure the required environment variables for the API to run locally. You can simply rename the file to `.env` and everything should work fine.

If needed do change the `DYNAMODB_LOCAL_ENDPOINT`/ `DYNAMODB_LOCAL_REGION` or the `DYNAMODB_LOCAL_SENSORS_TABLE_NAME` if you have done any custom changes.

You will also find two more environment variables related to integration tests. These can be configured after deploying the API to the `dev` stage. Then you would have the necessary API URL to configure as the `INTEGRATION_TEST_BASE_URL` variable. The `INTEGRATION_TEST_STAGE` can remain as `dev`.

After configuring the `.env` file the project is now ready to run locally. Run the project locally using the following command. (Uses [serverless-offline](https://www.serverless.com/plugins/serverless-offline) plugin)

```
npm start
```

### Deployment

To deploy the API to the `dev` stage run the following command.

```
npm run deploy-dev
```

This should deploy the API to the dev stage and produce an API gateway URL which makes the REST API ready-to-use. Refer to the [CI/CD](https://github.com/keshav1002/iot-device-management-api?tab=readme-ov-file#cicd) section for streamlined deployments.

## Scenario: Smart Building Temperature Monitoring System

### Overview

Imagine a smart building equipped with various IoT devices that monitor temperature levels in critical areas such as ventilation shafts, corridors, and server rooms. These devices are essential for maintaining optimal conditions and ensuring the safety and efficiency of the building's operations.

### Use Cases

1. Device Registration

   - **Situation:** The building management wants to monitor the temperature in the ventilation shaft and corridors. They install a Honeywell Temperature Sensor in the ventilation shaft and an Arduino Temperature Sensor in the corridor.

   - **API Action:** The management registers these devices using the API and save metadata such as the device name and its location.

2. Temperature Monitoring

   - **Situation:** The devices continuously monitor temperature levels. Every few seconds, they send temperature readings to the system. Each reading includes a timestamp and the temperature value.

   - **API Action:** The devices send their readings to the system using the API. Each reading includes a timestamp and the temperature value.

3. Anomaly Detection

   - **Situation:** Occasionally, the Honeywell Temperature Sensor detects a sharp increase in temperature in the ventilation shaft, rising from 41°C to 70°C within seconds. This spike could indicate an overheating issue. Upon detecting this anomaly, the device sends an error status labeled as "High" using the API. Similarly, the Arduino Temperature Sensor in the corridor records a sudden drop in temperature to 20°C, which is below the expected range and thereby resulting in an error status of "Low".

   - **API Action:** The management would like to identify what devices have logged an error status as High/ Low at any given time. Conversely, they would also like to see for a particular device what are the error readings logged in the system.

4. Data Retrieval:

   - **Situation:** The management wants to review the historical temperature data for analysis and reporting.

   - **API Action:** Ideally they would want to retrieve both Device details and its readings using the same API call. This makes it easier for them to understand and visualize. Further they would also like to see the latest readings from a given device. (Ex:- Last 10 readings) This helps to keep track of devices easier.

5. Device Maintenance and Updates:

   - **Situation:** Over time, the management decides to replace the Arduino Temperature Sensor with a more advanced model.

   - **API Action:** The device information needs to be updated in the system.

6. Device Decommissioning:

   - **Situation**: Eventually, a device might need to be removed from the system, such as when a sensor is retired or relocated.

   - **API Action:** The management deletes the device record using the API, ensuring the system's data remains current and relevant.

This API-driven solution enables real-time monitoring and management of IoT devices in a smart building environment. It helps ensure that critical areas maintain optimal temperature levels and allows the building management to respond quickly to any anomalies, enhancing safety and operational efficiency.

## API Documentation

There are 4 main CRUD API endpoints to facilitate the scenario of adding, retrieving, updating and deleting a IoT sensor device. Furthermore, there are 4 more API endpoints to facilitate some of the other use cases mentioned above.

The full documentation has been published as a Postman collection which can be found [here](https://documenter.getpostman.com/view/2375834/2sA3s9Docr). You can also import the collection and get started yourself.

## High-Level Architecture and Database Design

The diagram below (_Figure 1_) shows the high-level architecture of the developed REST API. Since it utilizes the serverless framework the API is built using Lambda functions and API gateway. Each API endpoint corresponds to a particular lambda function. The [Lambda proxy integration](https://docs.aws.amazon.com/apigateway/latest/developerguide/set-up-lambda-proxy-integrations.html) is used to connect with the API gateway.

Each lambda function is also connected to its own Cloudwatch log group. The application logs as well as metrics will be captured here. Furthermore permissions for each lambda function to write to Cloudwatch as well read/write to the DynamoDB database and its indexes is configured using IAM roles. Each function has its own role. This is configured using the [serverless-iam-roles-per-function plugin](https://www.serverless.com/plugins/serverless-iam-roles-per-function). Having a role for each function ensures that only the fine-grained necessary permissions are granted for each lambda function thereby enhancing security.

![High-Level Architecture](https://github.com/keshav1002/iot-device-management-api/blob/main/.github/images/iot-api-architecture.jpg?raw=true 'High-Level Architecture')

_Figure 1_

The main database utilized for this API is DynamoDB. DynamoDB is a serverless, NoSQL, fully managed database with single-digit millisecond performance at any scale. This makes it an idea choice for scenarios such as IoT sensors, as they tend to produce high volumes of data and require highly performant databases for optimized read and write operations.

The [single-table design](https://aws.amazon.com/blogs/compute/creating-a-single-table-design-with-amazon-dynamodb/) philosophy of DynamoDB has been utilized to design the table for this scenario. A single-table called "sensors" will be utilized to store both device metadata and its readings. The main reason for using a single table in DynamoDB is to retrieve multiple, heterogenous item types using a single request. This is beneficial in this scenario as we can retrieve both device metadata and readings from the sensor in one-shot using a single query. The design along with sample data can be found in the diagram below (_Figure 2_).

![DynamoDB Table Design](https://github.com/keshav1002/iot-device-management-api/blob/main/.github/images/dynamodb-table-design.png?raw=true 'DynamoDB Table Design')

_Figure 2_

### Database Keys and Indexes

- **Composite Primary Key:** The primary key for this table has been designed as a composite key consisting of the partition key and sort key. The partition key is designed to contain a `DEVICE` label followed by a delimiter `#` and a unique ID value. The sort key is heterogenous. It could either have a `DETAILS` label and the device ID to denote that this record contains device metadata. or it can contain a `READING` label and the timestamp to denote that this record contains an actual reading. Having such keys with delimiters allows for easier querying and to differentiate between record types.

- **Global Secondary Index (GSI):** A GSI `ReadingsByError` has been configured on the table using the `ErrorStatus` attribute as the partition key. This enables queries such as figuring out the devices that have been emitting a "High" error status easily ([Use Case 3](https://github.com/keshav1002/iot-device-management-api/tree/main?tab=readme-ov-file#use-cases)), without having the need to performing complex scans and filters on the main table.

- **Local Secondary Index (LSI):** A LSI `ErrorsByDevice` has also been configured. LSIs are useful in scenarios where the same partition key as the main table can be used but a different sort key is required. In this scenario the `ErrorStatus` attribute has been configured as a sort key along with `PK` as the partition key for the LSI. This allows for faster queries for scenarios such as figuring out how many errors have been logged by a particular device in the system ([Use Case 3](https://github.com/keshav1002/iot-device-management-api/tree/main?tab=readme-ov-file#use-cases)).

## Unit Test

Unit tests have been configured for this project using [jest](https://jestjs.io/). The local DynamoDB server is utilized to perform the unit tests. Thereby the unit tests can only be run locally as of now. It tests if the core logic in the controllers are working as intended.

To perform the unit tests execute the following command,

```
npm run test:unit
```

You can also run the following command to check the coverage.

```
npm run test:unit:coverage
```

![Unit Test Coverage](https://github.com/keshav1002/iot-device-management-api/blob/main/.github/images/unit-test-coverage.png?raw=true 'Unit Test Coverage')

_Figure 3_

## Integration Test

Integration tests have also been configured using the jest library along with the help of [supertest](https://www.npmjs.com/package/supertest) library. The integration tests checks if the deployed endpoints work as expected. This validates if the integrations between API Gateway, Lambda and DynamoDB are working as expected end-to-end.

To perform the integration tests execute the following command,

```
npm run test:integration
```

## CI/CD

The CI/CD for this project has been configured using [Github Actions](https://github.com/keshav1002/iot-device-management-api/actions). The deployment workflow file can be found under `.github/workflows/deploy.yml`.

Every time theres a push to the main branch the workflow is triggered and it deploys to the `dev` stage automatically. There's also a manual approval dispatch event that promotes the deployment to the `prod` stage.

The manual approval is added to prevent any accidental deployments to prod. Integration tests have been configured within the pipeline. The tests run automatically after the deployment. After the deployment to `dev` and if the integration tests for that environment passes, then its a good indication to promote that deployment to `prod`. Manual testing can also be done before promoting to `prod`. A screenshot of the CI/CD pipeline running in the `dev` stage can be seen below. (_Figure 4_)

![CI/CD Screenshot](https://github.com/keshav1002/iot-device-management-api/blob/main/.github/images/ci-cd-screenshot.png?raw=true 'CI/CD Screenshot')

_Figure 4_

There are 3 main secret values that need to be configured in your Github repository in order for the Github Actions pipeline to work correctly. If you're unsure how to configure the secrets you can follow [this guide](https://docs.github.com/en/actions/security-for-github-actions/security-guides/using-secrets-in-github-actions).

1. `AWS_ACCESS_KEY_ID` - Required to deploy to your AWS account. Ensure only the necessary permissions are granted and the credentials are generated. You can follow [this guide](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html) to generate the access and secret access keys.
2. `AWS_SECRET_ACCESS_KEY` - Will be generated along with the above key. Both are required.
3. `SERVERLESS_ACCESS_KEY` - Since Serverless Framework v4 a serverless access key is required to do deployments using the Serverless CLI. You can follow [this guide](https://www.serverless.com/framework/docs/guides/dashboard/cicd/running-in-your-own-cicd#create-an-access-key-in-the-serverless-framework-dashboard) to generate your own serverless access key.

## Sonarcloud

Sonarcloud has also been configured for this repo, the analysis can be accessed using this [link](https://sonarcloud.io/summary/overall?id=keshav1002_iot-device-management-api). The overall quality gate indicates a pass status with no major issues.

![Sonarcloud Screenshot](https://github.com/keshav1002/iot-device-management-api/blob/main/.github/images/sonarcloud-screenshot.png?raw=true 'Sonarcloud Screenshot')

_Figure 5_

## Logs

A custom logger is implemented within the API to record useful metrics. The logger was implemented using the [@aws-lambda-powertools/logger](https://www.npmjs.com/package/@aws-lambda-powertools/logger) library.

The logger records DynamoDB response times for all APIs, this leads to useful insights through queries in the Cloudwatch Logs Insights portal. A sample query can be found below.

```
fields @timestamp, handler, cold_start, duration
| filter service='iot-device-management' and message='Result'
| sort @timestamp desc
| stats min(duration), avg(duration), max(duration) by handler

```

![Cloudwatch Logs Screenshot](https://github.com/keshav1002/iot-device-management-api/blob/main/.github/images/cloudwatch-logs-screenshot.png?raw=true 'Cloudwatch Logs Screenshot')

_Figure 6_

The logger implementation can be extended to record varied custom metrics in the future.
