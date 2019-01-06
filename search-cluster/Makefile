

ifndef env
# $(error env is not set)
	env ?= dev
endif

ifdef CONFIG
	include ../$(CONFIG)
	export
else
	include ../config.$(env)
	export
endif

# STACK_PREFIX is custom to your deployment and should be the same for all Antiope Stacks
ifndef STACK_PREFIX
	$(error STACK_PREFIX is not set)
endif

ifndef BUCKET
	$(error BUCKET is not set)
endif

INDICES=resources_cloudtrail resources_dx_connections resources_dx_gw resources_dx_vif resources_ec2_eni resources_ec2_instance \
		resources_ec2_vpc resources_ecr_repositories resources_ecs_cluster resources_es_domain resources_iam_role \
		resources_iam_user resources_kms_key resources_lambda resources_route53_domains resources_route53_hostedzone \
		resources_s3_bucket resources_secretsmanager resources_ec2_ami

ifndef version
	export version := $(shell date +%Y%b%d-%H%M)
endif

# Specific to this stack
export STACK_NAME=search-cluster
# Filename for the CFT to deploy
export STACK_TEMPLATE=cloudformation/SearchCluster-Template.yaml

# Name of the Zip file with all the function code and dependencies
export LAMBDA_PACKAGE ?= $(STACK_NAME)-lambda-$(version).zip

# The full name of the stack in Cloudformation. This must match the manifest file
export FULL_STACK_NAME=$(STACK_PREFIX)-$(env)-$(STACK_NAME)

# Name of the manifest file.
export manifest=cloudformation/$(FULL_STACK_NAME)-Manifest.yaml

# location in the Antiope bucket where we drop lambda-packages
export OBJECT_KEY=deploy-packages/$(LAMBDA_PACKAGE)

export COGNITO_STACK_NAME=$(STACK_PREFIX)-$(env)-cognito

# List of all the functions deployed by this stack. Required for "make update" to work.
FUNCTIONS = $(FULL_STACK_NAME)-ingest-s3

.PHONY: $(FUNCTIONS)

# Run all tests
test: cfn-validate
	cd lambda && $(MAKE) test

# Do everything
deploy: package upload cfn-deploy

clean:
	cd lambda && $(MAKE) clean
	cd scripts && $(MAKE) clean
	rm -f notification_template-$(FULL_STACK_NAME).json

#
# Cloudformation Targets
#

# Validate the template
cfn-validate: $(STACK_TEMPLATE)
	aws cloudformation validate-template --region $(AWS_DEFAULT_REGION) --template-body file://$(STACK_TEMPLATE)

# Deploy the stack
cfn-deploy: cfn-validate $(manifest)
	deploy_stack.rb -m $(manifest) pLambdaZipFile=$(OBJECT_KEY) pBucketName=$(BUCKET) pEnvironment=$(env)  --force

#
# Lambda Targets
#
package:
	cd lambda && $(MAKE) package

upload: package
	aws s3 cp lambda/$(LAMBDA_PACKAGE) s3://$(BUCKET)/$(OBJECT_KEY)

# # Update the Lambda Code without modifying the CF Stack
update: $(FUNCTIONS)
	cd lambda && $(MAKE) zipfile
	for f in $(FUNCTIONS) ; do \
	  aws lambda update-function-code --function-name $$f --zip-file fileb://lambda/$(LAMBDA_PACKAGE) ; \
	done

# Update one specific function. Called as "make fupdate function=<fillinstackprefix>-aws-inventory-ecs-inventory"
fupdate: package
	aws lambda update-function-code --function-name $(function) --zip-file fileb://lambda/$(LAMBDA_PACKAGE) ; \


enable-s3-events:
	$(eval QUEUEARN := $(shell aws --region $(AWS_DEFAULT_REGION) cloudformation describe-stacks --stack-name $(FULL_STACK_NAME) --query 'Stacks[0].Outputs[?OutputKey==`SearchIngestEventQueueArn`].OutputValue' --output text))
	cat notification_template.json | jq '.QueueConfigurations[0].QueueArn = "$(QUEUEARN)"' > notification_template-$(FULL_STACK_NAME).json
	aws --region $(AWS_DEFAULT_REGION) s3api put-bucket-notification-configuration --bucket $(BUCKET) --notification-configuration file://notification_template-$(FULL_STACK_NAME).json
	rm notification_template-$(FULL_STACK_NAME).json

enable-kibana-auth:
	$(eval DOMAIN := $(shell aws cloudformation describe-stacks --stack-name $(FULL_STACK_NAME) --query 'Stacks[0].Outputs[?OutputKey==`ClusterName`].OutputValue' --output text --region $(AWS_DEFAULT_REGION)))
	$(eval ROLE := $(shell aws cloudformation describe-stacks --stack-name $(FULL_STACK_NAME) --query 'Stacks[0].Outputs[?OutputKey==`ESCognitoRoleArn`].OutputValue' --output text --region $(AWS_DEFAULT_REGION)))
	$(eval USER_POOL_ID := $(shell aws cloudformation describe-stacks --stack-name $(COGNITO_STACK_NAME) --query 'Stacks[0].Outputs[?OutputKey==`CognitoUserPoolId`].OutputValue' --output text --region $(AWS_DEFAULT_REGION)))
	$(eval ID_POOL_ID := $(shell aws cloudformation describe-stacks --stack-name $(COGNITO_STACK_NAME) --query 'Stacks[0].Outputs[?OutputKey==`CognitoIdentityPoolId`].OutputValue' --output text --region $(AWS_DEFAULT_REGION)))
	aws --region $(AWS_DEFAULT_REGION) es update-elasticsearch-domain-config --domain-name $(DOMAIN) --cognito-options Enabled=true,UserPoolId=$(USER_POOL_ID),IdentityPoolId=$(ID_POOL_ID),RoleArn=$(ROLE)

script-deps:
	cd scripts && $(MAKE) deps

purge-logs:
	for f in $(FUNCTIONS) ; do \
	  aws --region $(AWS_DEFAULT_REGION) logs delete-log-group --log-group-name /aws/lambda/$$f ; \
	done

expire-logs:
	for f in $(FUNCTIONS) ; do \
	  aws --region $(AWS_DEFAULT_REGION) logs put-retention-policy --log-group-name /aws/lambda/$$f --retention-in-days 5 ; \
	done

indices:
	for i in $(INDICES) ; do \
	  ./scripts/create_index.py --domain $(STACK_PREFIX)-$(env) --index $$i --mapping_dir mappings --delete ; \
	done

index:
	./scripts/create_index.py --domain $(STACK_PREFIX)-$(env) --index $(index) --mapping_dir mappings --delete

deploy-all: deploy script-deps enable-s3-events indices expire-logs enable-kibana-auth