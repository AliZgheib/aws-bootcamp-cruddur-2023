# Week 1 — App Containerization

## Required Homeworks/Tasks

### Running Cruddur without Docker

1. Set up the backend:

Navigate into the correct directory:
```
cd backend-flask
```
Install the required dependencies based on requirements.txt
```
pipe3 install -r requirements.txt
```
Add the necessary environment variables
```
export BACKEND_URL=*
export FRONTEND_URL=*
```
Run flask backend server
```
python -m flask run --debug --host=0.0.0.0 --port=4567
```

2. Set up the frontend:

Navigate into the correct directory:
```
cd frontend-react-js
```
Install the required dependencies based on package.json
```
npm install
```
Run the react application
```
npm run start
```

### Running Cruddur using Dockerfile

1. Step1
2. Step2

### Running Cruddur using Docker compose

### Adding DynamoDB Local and Postgres

#### Postgres

#### DynamoDB Local

#### Volumes

## Homework Challenges

### Running Cruddur outside of Gitpod / Codespaces ( Docker Desktop )

### Pushing docker images to DockerHub

### Pushing docker images to AWS ECR
