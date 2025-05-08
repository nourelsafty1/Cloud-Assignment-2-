# Cloud-Assignment-2-Event Driven Order System
## Step 1: DynamoDB Table Setup
1. **Navigate to DynamoDB Console**:
   - Go to [AWS DynamoDB Console](https://console.aws.amazon.com/dynamodbv2).
   - Click **Create table**.
2. **Configure Table**:
   - **Table name**: `Orders`
   - **Partition key**: `orderId` (Type: String)
   - **Add other attributes** (Optional): `userId`, `itemName`, `quantity`, `status`, `timestamp`.
   - Click **Create table**.

---

## Step 2: SNS Topic Creation
1. **Navigate to SNS Console**:
   - Go to [AWS SNS Console](https://console.aws.amazon.com/sns/v3).
   - Click **Create topic**.
2. **Configure Topic**:
   - **Type**: Standard
   - **Name**: `OrderTopic`
   - Click **Create topic**.
3. **Add SQS Subscription**:
   - Select the topic → **Subscriptions** → **Create subscription**.
   - **Protocol**: AWS SQS
   - **Endpoint**: Select `OrderQueue` (will be created in Step 3).
   - Click **Create subscription**.

---

## Step 3: SQS Queue Setup
1. **Navigate to SQS Console**:
   - Go to [AWS SQS Console](https://console.aws.amazon.com/sqs).
   - Click **Create queue**.
2. **Configure Queue**:
   - **Type**: Standard
   - **Name**: `OrderQueue`
   - **Dead-Letter Queue (DLQ)**:
     - Under **Dead-letter queue**, select **Enable**.
     - **Queue name**: `OrderDLQ`
     - **Maximum receives**: `3`.
   - Click **Create queue**.
3. **Subscribe to SNS Topic**:
   - Select `OrderQueue` → **Subscription** → **Subscribe to Amazon SNS topic**.
   - Choose `OrderTopic` → **Save**.

---

## Step 4: Lambda Function Deployment
1. **Navigate to Lambda Console**:
   - Go to [AWS Lambda Console](https://console.aws.amazon.com/lambda).
   - Click **Create function**.
2. **Configure Function**:
   - **Function name**: `ProcessOrderFunction`
   - **Runtime**: Python 3.9
   - Click **Create function**.
3. **Add SQS Trigger**:
   - Go to **Configuration** → **Triggers** → **Add trigger**.
   - Select **SQS** → Choose `OrderQueue` → **Save**.
4. **Paste Lambda Code**:
   - Replace the default code with [lambda_function.py](#lambda-function-code).
   - Click **Deploy**.

---

## Step 5: Testing the Flow
1. **Publish a Test Message to SNS**:
   - Go to `OrderTopic` → **Publish message**.
   - Paste this JSON:
     ```json
     {
       "orderId": "O1234", 
       "userId": "U123", 
       "itemName": "Laptop", 
       "quantity": 1, 
       "status": "new", 
       "timestamp": "2025-05-03T12:00:00Z"
     }
     ```
   - Click **Publish**.
2. **Verify Execution**:
   - **SQS**: Check `OrderQueue` for the message (poll if needed).
   - **DynamoDB**: Scan the `Orders` table for the new item.
   - **Lambda Logs**: View logs in CloudWatch under `/aws/lambda/ProcessOrder`.

---

# How Visibility Timeout and DLQ Were Useful in This Assignment  

## 1. Visibility Timeout  

### **Prevented Message Loss During Lambda Failures**  
- If `ProcessOrderFunction` crashed while saving to DynamoDB (e.g., due to throttling), the message temporarily disappeared from `OrderQueue` (visibility timeout) and reappeared after **30 seconds** for a retry.  
- **Example**: A transient DynamoDB error would cause Lambda to fail, but the message retried automatically.  

### **Avoided Duplicate Processing**  
- Ensured only **one Lambda instance** processed a message at a time, even if multiple consumers were polling the queue.  

---

## 2. Dead-Letter Queue (DLQ)  

### **Handled Poison Pills**  
- Messages with invalid data (e.g., `quantity: "two"` instead of `2`) failed processing repeatedly. After **3 attempts** (`maxReceiveCount`), they moved to `OrderDLQ` instead of cycling infinitely.  
- **Example**: A malformed test message would end up in `OrderDLQ`, letting you debug without blocking valid orders.  

### **Simplified Debugging**  
- You could inspect failed messages in `OrderDLQ` (via **AWS Console** or **CLI**) to identify issues like:  
  - Schema mismatches  
  - Missing fields  

---

## Why This Mattered for the Assignment  

1) **Resilience**: The system handled temporary failures (e.g., DynamoDB timeouts) gracefully.  

2) **Visibility**: You could verify the entire flow worked—even when errors occurred—by checking:  
- **SQS** (messages disappearing/reappearing during timeout).  
- **DLQ** (captured bad messages after retries).  

---

# Architecture Diagram
![Architecture Diagram drawio](https://github.com/user-attachments/assets/b85b4263-7372-4b34-a118-29345b4507a2)

