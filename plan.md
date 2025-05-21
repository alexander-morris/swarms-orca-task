# Swarms-Koii-Task Implementation Plan

## Architecture Overview

The system consists of three main components:
1. **Task Layer** (`task/src`) - Multiple instances
2. **Middle Server** (`middle-server`) - Single instance
3. **Docker Container** (`task/docker-container`) - One per task instance

## Message Passing Flow

```
  [Middle Server]
       ^   |
       |   | 1. Request jobs
       |   v
   [Task Layer] <---> [Docker Container]
       ^   |    2. Process    ^   |
       |   |       jobs       |   |
       |   v                  |   v
3. Return results   4. Return processed results
```

### Step-by-Step Flow with File References

1. **Job Acquisition**
   - Each Task instance requests work items ("jobs") from the Middle Server
   - **Key Files**:
     - `task/src/task/0-setup.ts` - Sets up a cron job that periodically checks for new tasks (lines 6-13)
     - `task/src/task/0-setup.ts` - Fetches tasks from middle server (lines 15-54)
     - `task/src/constant.ts` - Defines the middle server URL
     - `middle-server/src/index.ts` - Handles the `/fetch-todo` endpoint (lines 20-51)

2. **Job Processing**
   - Task passes jobs to its Docker Container via REST API
   - **Key Files**:
     - `task/src/task/0-setup.ts` - Calls the Docker container with job data (lines 41-46)
     - `task/src/orca.ts` - Manages communication with the Docker container (lines 13-53)
     - `task/docker-container/app.py` - Handles the job request via the `/task/<task_id>` endpoint (lines 41-68)
     - `task/docker-container/utils.py` - Contains helper functions for job processing (lines 26-35 for compute_fibonacci)

3. **Result Collection**
   - Docker Container returns results back to its parent Task instance
   - **Key Files**:
     - `task/docker-container/app.py` - Sends results back to task (lines 58-67)
     - `task/docker-container/utils.py` - Defines the submit_to_js_task function (lines 37-52)
     - `task/src/task/5-routes.ts` - Defines the `/submit-to-js` endpoint that receives results (lines 18-53)

4. **Result Submission**
   - Task submits the processed results back to the Middle Server
   - **Key Files**:
     - `task/src/task/5-routes.ts` - Processes results and submits them to the middle server (lines 36-43)
     - `task/src/utils/ipfs.ts` - Handles storing results in IPFS (lines 5-25)
     - `middle-server/src/index.ts` - Defines the `/post-todo-result` endpoint (lines 53-74)
     - `middle-server/src/utils/verifySignature.ts` - Verifies request signature from task

## K2 Submission (Future Implementation)

While executing the above workflow, each Task instance will also:
- Prepare submission data for K2
- Make periodic submissions to K2 network
- This functionality will be ignored during initial testing
- **Key Files**:
  - `task/src/task/2-submission.ts` - Will handle submissions to K2
  - `task/src/index.ts` - References the submission component (line 4)

## Testing Focus

The initial implementation will focus on:
- Ensuring proper message passing between all three layers
- Verifying each Task can correctly process jobs through its Docker Container
- Confirming results are correctly returned to the Middle Server

The system should handle multiple Task instances each with their own Docker Container, all communicating with a single Middle Server instance.

## Testing Flow with Single Schema Object

To verify the complete message passing pipeline, we will implement a testing flow using a single unified schema object that flows through all system components. This approach ensures data consistency and validates component interactions.

### Schema Object Definition

```json
{
  "id": "test-task-[timestamp]",
  "type": "ComputeTask",
  "input": {
    "operation": "fibonacci",
    "value": 20,
    "metadata": {
      "priority": "high",
      "requestedBy": "test-system"
    }
  },
  "status": "pending",
  "timestamps": {
    "created": "[ISO date]",
    "fetchedByTask": null,
    "processedByDocker": null,
    "completedAt": null
  },
  "result": null
}
```

### Testing Sequence

1. **Initialization** *(Middle Server)*
   ```bash
   # Navigate to middle-server directory
   cd middle-server
   
   # Install dependencies
   yarn install
   
   # Start the middle server with test mode enabled
   TEST_MODE=true yarn start
   ```

2. **Schema Registration** *(Middle Server)*
   - Use a REST client (Postman/curl) to POST the schema object to a new test endpoint
   - The middle server stores this in its queue for task nodes to fetch
   - Middle server timestamp updates `created` field

3. **Task Node Fetch** *(Task)*
   ```bash
   # Navigate to task directory
   cd task
   
   # Install dependencies
   yarn install
   
   # Start the task node with test mode
   PYTHON_COMMAND=python3 TEST_MODE=true yarn start
   ```
   - Task node will fetch the task from middle server
   - Middle server timestamp updates `fetchedByTask` field
   - Task logs should show receipt of the schema object

4. **Docker Container Processing** *(Docker Container)*
   - Task node passes schema object to Docker container via REST
   - Docker container processes the task (computes fibonacci)
   - Docker container timestamp updates `processedByDocker` field
   - Docker container adds result to schema object

5. **Result Return** *(Task → Middle Server)*
   - Docker container returns updated schema to task node
   - Task node adds final timestamp `completedAt`
   - Task node sends completed schema back to middle server
   - Middle server verifies and stores the final result

### Verification Points

To verify the pipeline is working correctly, check these key points:

1. **Middle Server Logs**
   - Task registration is recorded
   - Task distribution to requesting nodes is recorded
   - Final result is received and recorded

2. **Task Logs**
   - Successful task fetch from middle server
   - Successful forwarding to Docker container
   - Receipt of processed results from Docker
   - Successful submission back to middle server

3. **Docker Container Logs**
   - Receipt of task from task node
   - Processing results
   - Successful return of results to task node

4. **Schema Object State**
   - All timestamps are properly updated at each stage
   - Result field contains correct computation output
   - Status field is updated at each stage of the pipeline

### Common Issues and Troubleshooting

1. **Package.json Not Found**
   - Ensure you're in the correct directory when running yarn commands
   - Each component has its own package.json file:
     - `middle-server/package.json`
     - `task/package.json`

2. **Docker Container Communication Failures**
   - Check the container is running: `docker ps`
   - Verify host/port configuration in task/src/orca.ts
   - Ensure network connectivity between task and container

3. **Middle Server Connection Issues**
   - Verify middle server URL in task/src/constant.ts
   - Check firewall settings if running on different machines

This testing flow provides a comprehensive verification of the entire message passing pipeline using a single schema object that maintains state throughout the process.

## Starting the System

1. Start the Middle Server:
   ```
   cd middle-server
   yarn start
   ```

2. Start the Task with Docker Container:
   ```
   cd task
   PYTHON_COMMAND=python3 PYTHON_SERVER_PORT=8081 MIDDLE_SERVER_PORT=5002 yarn start
   ```

## Workflow Improvements

Based on the existing architecture and testing results, here are several improvements to enhance the development and testing workflow:

### 1. Directory Structure and Navigation

**Current Issue:** Commands run from the root directory fail with "Couldn't find a package.json file".

**Solution:** Create a root package.json with workspace definitions and cross-component scripts:

```json
// package.json in root directory
{
  "name": "swarms-koii-task",
  "version": "1.0.0",
  "private": true,
  "workspaces": [
    "task",
    "middle-server"
  ],
  "scripts": {
    "start:middle": "cd middle-server && yarn start",
    "start:task": "cd task && PYTHON_COMMAND=python3 PYTHON_SERVER_PORT=8081 MIDDLE_SERVER_PORT=5002 yarn start",
    "start:all": "concurrently \"yarn start:middle\" \"yarn start:task\"",
    "install:all": "yarn install && yarn workspaces run install",
    "test:pipeline": "cd scripts && node test-pipeline.js"
  },
  "devDependencies": {
    "concurrently": "^8.0.0"
  }
}
```

### 2. Docker Compose Integration

Add a docker-compose.yml file to manage all components together:

```yaml
# docker-compose.yml
version: '3'

services:
  middle-server:
    build:
      context: ./middle-server
    ports:
      - "5002:5002"
    environment:
      - PORT=5002
    volumes:
      - ./middle-server:/app
      - /app/node_modules

  task:
    build:
      context: ./task
    depends_on:
      - middle-server
    environment:
      - MIDDLE_SERVER_HOST=middle-server
      - MIDDLE_SERVER_PORT=5002
      - PYTHON_SERVER_HOST=localhost
      - PYTHON_SERVER_PORT=8081
    volumes:
      - ./task:/app
      - /app/node_modules
    # We intentionally don't expose ports here as each task instance
    # should be isolated in a normal environment
```

### 3. Environment Configuration

Create .env files for each component with sensible defaults and add .env.example files to the repo:

```
# task/.env.example
MIDDLE_SERVER_HOST=localhost
MIDDLE_SERVER_PORT=5002
PYTHON_SERVER_HOST=localhost
PYTHON_SERVER_PORT=8081
PYTHON_COMMAND=python3
```

```
# middle-server/.env.example
HOST=localhost
PORT=5002
```

### 4. Testing Automation Script

Create a Node.js script to automate the testing flow:

```javascript
// scripts/test-pipeline.js
const axios = require('axios');
const { execSync } = require('child_process');
const fs = require('fs');

async function runTest() {
  console.log('Starting test pipeline...');
  
  // Create the test schema
  const schema = {
    id: `test-task-${Date.now()}`,
    type: 'ComputeTask',
    input: {
      operation: 'fibonacci',
      value: 20,
      metadata: {
        priority: 'high',
        requestedBy: 'test-system'
      }
    },
    status: 'pending',
    timestamps: {
      created: new Date().toISOString(),
      fetchedByTask: null,
      processedByDocker: null,
      completedAt: null
    },
    result: null
  };
  
  // Save schema to file for reference
  fs.writeFileSync('test-schema.json', JSON.stringify(schema, null, 2));
  
  // Register schema with middle server
  try {
    const response = await axios.post('http://localhost:5002/register-test-task', schema);
    console.log('Schema registered with middle server:', response.data);
    
    // Now start the task to process it
    console.log('Starting task to process the schema...');
    console.log('Check task logs for progress. Test schema saved to test-schema.json');
  } catch (err) {
    console.error('Error in test pipeline:', err.message);
  }
}

runTest();
```

### 5. Development Workflow

Implement a streamlined development workflow:

1. **Setup**:
   ```bash
   # From root directory
   yarn install:all
   ```

2. **Running Components**:
   ```bash
   # Option 1: Start all components
   yarn start:all
   
   # Option 2: Start components individually
   yarn start:middle
   yarn start:task
   
   # Option 3: Docker Compose (recommended for production-like testing)
   docker-compose up
   ```

3. **Testing**:
   ```bash
   # Run automated test
   yarn test:pipeline
   ```

### 6. Monitoring and Debugging

Add monitoring endpoints to each component:

1. **Middle Server**: Add `/status` endpoint that shows:
   - Number of pending tasks
   - Number of completed tasks
   - System uptime

2. **Task**: Add `/health` endpoint that shows:
   - Connection status to middle server
   - Docker container status
   - Current task status

3. **Docker Container**: Add a monitoring endpoint that shows:
   - Current system resources usage
   - Processing statistics

### 7. Error Handling Improvements

1. Add retry mechanisms for network failures
2. Implement graceful degradation when components are unavailable
3. Add detailed logging with correlation IDs across components

By implementing these improvements, the workflow will be more robust, easier to use, and better suited for both development and testing environments.
