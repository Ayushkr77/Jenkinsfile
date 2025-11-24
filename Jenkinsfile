pipeline {
  agent any

  environment {
    REGION = 'us-east-1'              // change to your region
    STACK_NAME = 'lambda-dynamo-inline'
    LAMBDA_NAME = "my-lambda-dev"     // matches CFN naming
    ENV = 'dev'
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Deploy CloudFormation') {
      steps {
        sh '''
          # Ensure aws cli is present (EC2 should have it)
          if ! command -v aws >/dev/null 2>&1; then
            python3 -m pip install --user awscli
            export PATH=$HOME/.local/bin:$PATH
          fi
          export AWS_REGION=${REGION}
          export AWS_DEFAULT_REGION=${REGION}
          echo "Deploying CloudFormation..."
          aws cloudformation deploy --template-file template.yml \
            --stack-name ${STACK_NAME} \
            --parameter-overrides Env=${ENV} \
            --capabilities CAPABILITY_NAMED_IAM \
            --region ${REGION}
        '''
      }
    }

    stage('Smoke test: Invoke Lambda') {
      steps {
        sh '''
          export AWS_REGION=${REGION}; export AWS_DEFAULT_REGION=${REGION}
          aws lambda invoke --function-name ${LAMBDA_NAME} --payload '{"smoketest":true}' --cli-binary-format raw-in-base64-out /tmp/lambda_out.json --region ${REGION}
          cat /tmp/lambda_out.json
          # Extract id using python
          ID=$(python3 - <<'PY'
import json
d=json.load(open('/tmp/lambda_out.json'))
p=d.get('Payload') or d
if isinstance(p,str):
    try:
        inner=json.loads(p)
    except:
        inner={}
else:
    inner=p
b=inner.get('body')
if b:
    try:
        print(json.loads(b).get('id',''))
    except:
        print('')
else:
    print('')
PY
)
          echo "Extracted ID: $ID"
          if [ -z "$ID" ]; then
            echo "Failed to extract ID from Lambda output" >&2
            exit 1
          fi
          aws dynamodb get-item --table-name MyTable-${ENV} --key "{\"id\":{\"S\":\"${ID}\"}}" --region ${REGION}
        '''
      }
    }
  }

  post {
    success { echo "Pipeline succeeded" }
    failure { echo "Pipeline failed — inspect logs" }
  }
}
