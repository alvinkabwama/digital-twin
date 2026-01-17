# AI Digital Twin

An AI-powered "Digital Twin" web application that provides an interactive chat interface backed by AWS Bedrock models. The project includes a Next.js frontend, a FastAPI backend (packaged for AWS Lambda), and Terraform infrastructure to deploy the frontend, API (Lambda + API Gateway), and S3 memory storage.

- Frontend: Next.js React app (Tailwind CSS).
- Backend: FastAPI app that calls AWS Bedrock via boto3 and stores conversation memory locally or in S3.
- Infra: Terraform to provision S3, Lambda, API Gateway, CloudFront, and optional custom domain + ACM.

Contents
- Project overview
- Architecture (text)
- Setup & local development
- Deployment (Terraform / scripts)
- Environment variables
- API reference (endpoints, request/response)
- Important files and where to look
- Notes and troubleshooting

## Project overview
- **Purpose:** Provide an AI chat interface ("Digital Twin") for course or deployment guidance, with persistent conversation memory and model selection controlled via environment variables.
- **Key capabilities:**
  - Chat with an AI assistant using AWS Bedrock models.
  - Persist conversation history to local filesystem or S3.
  - Host frontend as a static site via S3 + CloudFront.
  - Serve backend as AWS Lambda behind API Gateway (or run locally with uvicorn).

## Architecture (text)
- User browser (`Next.js` frontend) ↔ API Gateway ↔ AWS Lambda (FastAPI packaged) ↔ AWS Bedrock
- Conversation memory stored in S3 (or local `memory/` during local dev)
- Frontend hosted in S3 + CloudFront

## Local development
1. Backend
   - Create a Python virtual environment and install dependencies from `backend/requirements.txt`.
   - Run locally:
     - From `backend/`: `uvicorn server:app --host 0.0.0.0 --port 8000`
   - Lambda handler (for packaging/deployment) is `backend/lambda_handler.py`.

2. Frontend
   - From `frontend/`:
     - Install: `npm install`
     - Dev: `npm run dev`
     - Build: `npm run build`
   - The frontend uses `NEXT_PUBLIC_API_URL` to determine the API base (defaults to `http://localhost:8000`).

3. Terraform (in `terraform/`)
   - Configure `terraform.tfvars` (project name, environment, domain options).
   - `terraform init && terraform apply` to provision AWS resources.
   - The Terraform config creates:
     - S3 bucket for memory and S3 bucket for frontend website
     - Lambda function using `backend/lambda-deployment.zip`
     - API Gateway HTTP API with routes for `/`, `/chat`, `/health`, `/conversation/{session_id}`
     - CloudFront distribution for the frontend

## Deployment notes
- Packaging backend for Lambda:
  - A zip file `backend/lambda-deployment.zip` is expected by Terraform; `terraform/main.tf` references this for `aws_lambda_function.api`.
- Scripts:
  - See `scripts/` for `deploy.ps1` and `destroy.ps1` used for automating deploy/destroy steps.
- IAM:
  - Terraform attaches `AWSLambdaBasicExecutionRole`, `AmazonBedrockFullAccess`, and `AmazonS3FullAccess` to the Lambda role to allow Bedrock and S3 access. For production, restrict to least privilege.

## Environment variables
- Backend (via `.env` or Lambda environment variables):
  - `CORS_ORIGINS` — CSV of allowed origins (default: `http://localhost:3000`)
  - `DEFAULT_AWS_REGION` — AWS region for bedrock/runtime client (default `us-east-1`)
  - `BEDROCK_MODEL_ID` — Bedrock model ID to use (e.g. `amazon.nova-lite-v1:0`)
  - `USE_S3` — `"true"` or `"false"`. If `true`, conversation memory stored in S3.
  - `S3_BUCKET` — S3 bucket name for memory (set by Terraform)
  - `MEMORY_DIR` — local directory for memory files (default `../memory`)
- Frontend:
  - `NEXT_PUBLIC_API_URL` — Base URL for API calls. Defaults to `http://localhost:8000` for local dev.

## API reference
The backend implementation lives at `backend/server.py`. Key endpoints:

- Root
  - `GET /`
  - Response JSON:
    - `message`: informational string
    - `memory_enabled`: boolean
    - `storage`: `"S3"` or `"local"`
    - `ai_model`: selected Bedrock model

- Health
  - `GET /health`
  - Response JSON:
    - `status`: `"healthy"` (or other status)
    - `use_s3`: boolean
    - `bedrock_model`: model id in use

- Chat
  - `POST /chat`
  - Request JSON:
    - `message` (string) — user message
    - `session_id` (string, optional) — existing session id to continue a conversation; server generates one if omitted
  - Response JSON:
    - `response` (string) — assistant reply
    - `session_id` (string) — session id to use for the conversation
  - Behavior:
    - Loads conversation history (local file or S3, depending on `USE_S3`).
    - Calls AWS Bedrock via boto3 with a system prompt and recent conversation history.
    - Appends both user and assistant messages to persistent storage.

- Get conversation
  - `GET /conversation/{session_id}`
  - Response JSON:
    - `session_id`
    - `messages`: array of `{ role, content, timestamp }`

Code references (examples in repo):
```text
```165:172:backend/server.py
@app.get("/")
async def root():
    return {
        "message": "AI Digital Twin API (Powered by AWS Bedrock)",
        "memory_enabled": True,
        "storage": "S3" if USE_S3 else "local",
        "ai_model": BEDROCK_MODEL_ID
    }
```
```

## Frontend usage
- Component: `frontend/components/twin.tsx` implements the chat UI and posts to `/chat`.
- The frontend sends a POST to `${NEXT_PUBLIC_API_URL || 'http://localhost:8000'}/chat` with `{ message, session_id }` and displays the returned `response` and `session_id`.

## Infrastructure (Terraform)
- `terraform/main.tf` provisions:
  - S3 buckets: memory bucket and frontend site bucket
  - Lambda function: uses `backend/lambda-deployment.zip`
  - API Gateway HTTP API: routes mapped to Lambda for `GET /`, `GET /health`, `POST /chat`, `GET /conversation/{session_id}`
  - CloudFront distribution and optional custom domain configuration with ACM + Route53

Snippet:
```text
```122:141:terraform/main.tf
resource "aws_lambda_function" "api" {
  filename         = "${path.module}/../backend/lambda-deployment.zip"
  function_name    = "${local.name_prefix}-api"
  role             = aws_iam_role.lambda_role.arn
  handler          = "lambda_handler.handler"
  runtime          = "python3.12"
  ...
  environment {
    variables = {
      CORS_ORIGINS     = ...
      S3_BUCKET        = aws_s3_bucket.memory.id
      USE_S3           = "true"
      BEDROCK_MODEL_ID = var.bedrock_model_id
    }
  }
}
```
```

## Key files
- Backend:
  - `backend/server.py` — FastAPI app with endpoints, Bedrock integration, memory handling.
  - `backend/context.py` — system prompt (imported by `server.py`).
  - `backend/lambda_handler.py` — `Mangum` adapter for Lambda.
  - `backend/lambda-deployment.zip` — packaged backend used by Terraform.
- Frontend:
  - `frontend/components/twin.tsx` — chat UI and API call logic.
  - `frontend/app/layout.tsx`, `frontend/app/page.tsx` — app shell / pages.
  - `frontend/package.json` — dev/build scripts and dependencies.
- Infra:
  - `terraform/main.tf` — core infra resources.
  - `scripts/` — `deploy.ps1` and `destroy.ps1`.

## Security & permissions
- Terraform currently grants broad Bedrock and S3 permissions to the Lambda role. Narrow policies to least privilege for production.
- Keep AWS credentials out of the repository and avoid committing `.env` files that include secrets.

## Troubleshooting
- If Bedrock access errors occur, verify IAM permissions and `BEDROCK_MODEL_ID` (region prefixes may be required).
- For CORS issues, ensure `CORS_ORIGINS` includes your frontend domain or configure `NEXT_PUBLIC_API_URL`.
- When running locally, ensure `uvicorn server:app` uses the same env vars the frontend expects.

## Dependencies (frontend)
- Notable dependencies in `frontend/package.json`:
```json
{
  "dependencies": {
    "lucide-react": "^0.562.0",
    "next": "16.1.2",
    "react": "19.2.3",
    "react-dom": "19.2.3"
  }
}
```

## Recommended next steps
- Add a `Makefile` or `package.json` script to build `backend/lambda-deployment.zip`.
- Harden IAM policies to least privilege (Bedrock and S3).
- Add automated tests for the API endpoints and frontend UI.

---

If you'd like, I can:
- Create a packaging script to build `backend/lambda-deployment.zip`.
- Add a short Quickstart section with commands to run the full stack locally.

