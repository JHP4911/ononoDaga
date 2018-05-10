# onondaga-e911

## Data

**UI**: https://s3.amazonaws.com/onondaga-e911-dev/index.html

**DynamoDB Tables**

- `onondaga-e911-all-dev`
- `onondaga-e911-closed-dev`
- `onondaga-e911-all-dev`

**Data Dumps**:

```
aws s3 ls s3://onondaga-e911-dev/all/
aws s3 ls s3://onondaga-e911-dev/closed/
aws s3 ls s3://onondaga-e911-dev/pending/
```

Scrape Onondaga county's computer aided dispatch (CAD) E911 events: http://wowbn.ongov.net/CADInet/app/events.jsp

## Caveats

- **Unique Ids**: The majority of issues with the data stem from the fact that there is no unique id per row. To fix this, we create a SHA1 hash of all of the text (**excluding the timestamp**).
	- The primary key for the `all` and `closed` tables is `{timestamp}_{hash}`. The timestamp for the `pending` table is `{insertion_date}_{hash}`.
	- Keep in mind that the exact same row data (consisting of agency, address, cross streets etc.) can and very likely will happen multiple times so the `hash` by itself is not globally unique. The timestamp is not included as part of the `hash` to help map to the pending table where no timestamp is available.

- **Linking Data**: Data for pending/all/closed events is stored in three separate tables. Looking at a combination of the `hash` and the insertion timestamp in each table should help map the lifecycle of an event from pending to all (active) to closed. For example, the insertion timestamp of the `closed` table - the insertion timestamp of the `all` table should give the length of an event.

- **Pending Events**: The primary key of the pending events table is `{insertion_date}_{hash}`. If an event is inserted near midnight (EST) and remains pending until the next day, a duplicate record will be added. Check the insertion timestamp to ensure there are no back to back records with the same `hash` and consecutive insertion dates.

- **Dynamo/S3**: It can be expensive/slow (without some [finagling](./dynamodb.md)) to download the entire database directly off of DynamoDB when it gets larger (in about a year). This is why the daily S3 dumps are available. See the [Export All Data](#export-all-data) section for more details.

## Quickstart

1. Install dependencies.

	Must have node and python3.6 installed.

	```
	brew install jq
	pip3 install awscli
	pip3 install pipenv
	pipenv install
	npm install -g serverless
	npm install -g @mapbox/dyno
	npm install
	```

2. Set up AWS Credentials.

	You have two options for AWS Credentials: root user (easy, less secure) or creating IAM roles (slightly more work, more secure).

	To get the root AWS credentials of your account, go here: https://console.aws.amazon.com/iam/home?#security_credential

	To instead create a specific IAM user, go here: https://console.aws.amazon.com/iam/home?#/users You will need to allow the user to create lambdas, dynamodb tables, and log groups.

	Once you have the aws access key id and aws secret access key, this command below will help you set up the `default` profile for AWS:

	```
	aws configure
	```

	Run `cat ~/.aws/credentials` to ensure the credentials are set up properly. Make sure you have the `region` set.

	The output should look similar to:

	```
	[default]
	aws_access_key_id = xxx
	aws_secret_access_key = xxx
	region = us-east-1
	```

	For help, see: https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html

3. Create `.env` file.

	```
	touch .env
	```

	If you are using the `default` aws role, just leave the file blank. Otherwise, set: `AWS_PROFILE=xyz` where `xyz` is the name of your profile.

4. Deploy the stack with serverless.

	`pipenv shell` will activate a virtualenv and load your environment variables. `serverless deploy` will create the stack on AWS.

	```
	pipenv shell
	serverless deploy
	```

	To deploy to production add `--stage prod` to the deploy command.

## Checking Logs

Function names are `scrape_all` or `scrape_closed`.

```
serverless logs --function scrape_all --tail
```

## Running locally

```
pipenv shell # activate python virtualenv
serverless invoke local --function scrape_all
```

## Deleting everything

For `dev` stage:

```
aws s3 rm --recursive s3://onondaga-e911-dev
serverless remove
```

## Export Data

The names of tables are `onondaga-e911-(all|closed)-(dev|prod)`. For example `onondaga-e911-all-dev`.

### Export Single Date

This command exports data from "2018-04-24".

```
aws dynamodb query --table-name onondaga-e911-all-dev --key-condition-expression "#d = :date" --expression-attribute-values '{":date": {"S":"2018-04-24"}}' --expression-attribute-names '{"#d":"date"}' --return-consumed-capacity TOTAL | jq -f filter.jq
```

### Export All Data

Please note that the database is provisioned with 1 read capacity unit (RCU). You may want to increase this if you are exporting the entire database. Check [dynamo.md](./dynamo.md) for more details.

```
dyno scan us-east-1/onondaga-e911-all-dev | jq -f filter.jq
dyno scan us-east-1/onondaga-e911-closed-dev | jq -f filter.jq
```

You can pipe this into a file to save the data. This pipes the output into `onondaga-e911-all-dev.json` and `onondaga-e911-closed-dev.json` respectively.

```
dyno scan us-east-1/onondaga-e911-all-dev | jq -Mcf filter.jq > onondaga-e911-all-dev.json
dyno scan us-east-1/onondaga-e911-closed-dev | jq -Mcf filter.jq > onondaga-e911-closed-dev.json
```
