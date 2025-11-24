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
          # Ensure aws cli is present
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
          export AWS_REGION=${REGION}
          export AWS_DEFAULT_REGION=${REGION}

          # Invoke Lambda
          aws lambda invoke --function-name ${LAMBDA_NAME} \
            --payload '{"smoketest":true}' \
            --cli-binary-format raw-in-base64-out \
            /tmp/lambda_out.json --region ${REGION}

          cat /tmp/lambda_out.json

          # Extract ID safely
          ID=$(python3 - <<'PY'
import json
try:
    d = json.load(open('/tmp/lambda_out.json'))
    p = d.get('Payload') or d
    if isinstance(p, str):
        try:
            p = json.loads(p)
        except:
            p = {}
    body = p.get('body')
    if body:
        try:
            print(json.loads(body).get('id',''))
        except:
            print('')
    else:
        print('')
except Exception:
    print('')
PY
)
          echo "Extracted ID: $ID"

          if [ -z "$ID" ]; then
            echo "Failed to extract ID from Lambda output" >&2
            exit 1
          fi

          # Correctly format JSON for DynamoDB get-item
          KEY_JSON=$(printf '{"id":{"S":"%s"}}' "$ID")
          aws dynamodb get-item --table-name MyTable-${ENV} --key "$KEY_JSON" --region ${REGION}
        '''
      }
    }
  }

  post {
    success { echo "Pipeline succeeded" }
    failure { echo "Pipeline failed — inspect logs" }
  }
}
