# Booking Service with AWS Lambda in Go

I'll redesign the booking service using Go (Golang) for better performance and cold start characteristics. Here's a complete implementation:

## 1. Project Structure

```
booking-service/
├── go.mod
├── main.go
├── handlers/
│   ├── create_booking.go
│   ├── get_booking.go
│   ├── update_booking.go
│   ├── cancel_booking.go
│   └── notifications.go
├── models/
│   └── booking.go
├── db/
│   └── dynamodb.go
└── events/
    └── websocket.go
```

## 2. Core Implementation

### `main.go` (Entry Point)
```go
package main

import (
	"context"
	"log"
	
	"github.com/aws/aws-lambda-go/lambda"
	"github.com/aws/aws-sdk-go-v2/config"
	
	"booking-service/handlers"
	"booking-service/db"
)

var (
	dynamoClient *db.DynamoDBClient
)

func init() {
	// Initialize AWS SDK
	cfg, err := config.LoadDefaultConfig(context.TODO())
	if err != nil {
		log.Fatalf("failed to load AWS config: %v", err)
	}
	
	dynamoClient = db.NewDynamoDBClient(cfg)
}

func main() {
	lambda.Start(handlers.CreateBookingHandler)
}
```

### `models/booking.go`
```go
package models

import "time"

type Booking struct {
	BookingID   string    `json:"bookingId" dynamodbav:"bookingId"`
	UserID      string    `json:"userId" dynamodbav:"userId"`
	ServiceID   string    `json:"serviceId" dynamodbav:"serviceId"`
	ServiceName string    `json:"serviceName" dynamodbav:"serviceName"`
	DateTime    time.Time `json:"dateTime" dynamodbav:"dateTime"`
	Status      string    `json:"status" dynamodbav:"status"` // confirmed, cancelled, completed
	CreatedAt   time.Time `json:"createdAt" dynamodbav:"createdAt"`
	UpdatedAt   time.Time `json:"updatedAt" dynamodbav:"updatedAt"`
}

type CreateBookingRequest struct {
	UserID      string    `json:"userId"`
	ServiceID   string    `json:"serviceId"`
	ServiceName string    `json:"serviceName"`
	DateTime    time.Time `json:"dateTime"`
}

type BookingResponse struct {
	BookingID string    `json:"bookingId"`
	Status    string    `json:"status"`
	DateTime  time.Time `json:"dateTime"`
}
```

### `db/dynamodb.go`
```go
package db

import (
	"context"
	"log"

	"github.com/aws/aws-sdk-go-v2/aws"
	"github.com/aws/aws-sdk-go-v2/service/dynamodb"
	"github.com/aws/aws-sdk-go-v2/service/dynamodb/types"
)

type DynamoDBClient struct {
	client    *dynamodb.Client
	tableName string
}

func NewDynamoDBClient(cfg aws.Config) *DynamoDBClient {
	return &DynamoDBClient{
		client:    dynamodb.NewFromConfig(cfg),
		tableName: "BookingsTable",
	}
}

func (d *DynamoDBClient) CreateBooking(ctx context.Context, booking *models.Booking) error {
	item, err := attributevalue.MarshalMap(booking)
	if err != nil {
		return err
	}

	_, err = d.client.PutItem(ctx, &dynamodb.PutItemInput{
		TableName: aws.String(d.tableName),
		Item:      item,
	})
	
	return err
}

func (d *DynamoDBClient) GetBooking(ctx context.Context, bookingID string) (*models.Booking, error) {
	result, err := d.client.GetItem(ctx, &dynamodb.GetItemInput{
		TableName: aws.String(d.tableName),
		Key: map[string]types.AttributeValue{
			"bookingId": &types.AttributeValueMemberS{Value: bookingID},
		},
	})
	
	if err != nil {
		return nil, err
	}
	
	if result.Item == nil {
		return nil, nil
	}
	
	var booking models.Booking
	err = attributevalue.UnmarshalMap(result.Item, &booking)
	return &booking, err
}
```

## 3. Lambda Handlers

### `handlers/create_booking.go`
```go
package handlers

import (
	"context"
	"encoding/json"
	"log"
	"time"

	"github.com/aws/aws-lambda-go/events"
	"github.com/google/uuid"

	"booking-service/db"
	"booking-service/models"
	"booking-service/events"
)

func CreateBookingHandler(ctx context.Context, request events.APIGatewayProxyRequest) (events.APIGatewayProxyResponse, error) {
	var req models.CreateBookingRequest
	if err := json.Unmarshal([]byte(request.Body), &req); err != nil {
		return events.APIGatewayProxyResponse{
			StatusCode: 400,
			Body:       `{"error": "invalid request body"}`,
		}, nil
	}

	// Validate booking time
	if req.DateTime.Before(time.Now()) {
		return events.APIGatewayProxyResponse{
			StatusCode: 400,
			Body:       `{"error": "booking time must be in the future"}`,
		}, nil
	}

	booking := &models.Booking{
		BookingID:   uuid.New().String(),
		UserID:      req.UserID,
		ServiceID:   req.ServiceID,
		ServiceName: req.ServiceName,
		DateTime:    req.DateTime,
		Status:      "confirmed",
		CreatedAt:   time.Now(),
		UpdatedAt:   time.Now(),
	}

	if err := db.DynamoClient.CreateBooking(ctx, booking); err != nil {
		log.Printf("failed to create booking: %v", err)
		return events.APIGatewayProxyResponse{
			StatusCode: 500,
			Body:       `{"error": "failed to create booking"}`,
		}, nil
	}

	// Send notifications
	go events.SendBookingNotification(booking) // Async notification
	go events.UpdateDashboard(booking)        // Real-time dashboard update

	response := models.BookingResponse{
		BookingID: booking.BookingID,
		Status:    booking.Status,
		DateTime:  booking.DateTime,
	}

	body, _ := json.Marshal(response)
	return events.APIGatewayProxyResponse{
		StatusCode: 201,
		Body:       string(body),
	}, nil
}
```

## 4. Notification System

### `events/notifications.go`
```go
package events

import (
	"context"
	"encoding/json"
	"log"

	"github.com/aws/aws-sdk-go-v2/aws"
	"github.com/aws/aws-sdk-go-v2/service/sns"
	"github.com/aws/aws-sdk-go-v2/service/apigatewaymanagementapi"

	"booking-service/models"
)

func SendBookingNotification(booking *models.Booking) {
	// Initialize SNS client
	cfg, err := config.LoadDefaultConfig(context.TODO())
	if err != nil {
		log.Printf("failed to load AWS config: %v", err)
		return
	}

	client := sns.NewFromConfig(cfg)

	message := map[string]interface{}{
		"default": "Booking notification",
		"email":   "Your booking for " + booking.ServiceName + " is confirmed",
		"sms":     "Booking confirmed: " + booking.ServiceName,
	}

	messageBytes, _ := json.Marshal(message)

	_, err = client.Publish(context.TODO(), &sns.PublishInput{
		Message:          aws.String(string(messageBytes)),
		TopicArn:         aws.String("arn:aws:sns:us-east-1:123456789012:BookingNotifications"),
		MessageStructure: aws.String("json"),
	})

	if err != nil {
		log.Printf("failed to send notification: %v", err)
	}
}

func UpdateDashboard(booking *models.Booking) {
	// Initialize WebSocket client
	cfg, err := config.LoadDefaultConfig(context.TODO())
	if err != nil {
		log.Printf("failed to load AWS config: %v", err)
		return
	}

	client := apigatewaymanagementapi.NewFromConfig(cfg, func(o *apigatewaymanagementapi.Options) {
		o.EndpointResolver = apigatewaymanagementapi.EndpointResolverFromURL("https://{api-id}.execute-api.{region}.amazonaws.com/{stage}")
	})

	message := map[string]interface{}{
		"action": "bookingUpdate",
		"data":   booking,
	}

	messageBytes, _ := json.Marshal(message)

	// In a real implementation, you would look up connection IDs for the user
	_, err = client.PostToConnection(context.TODO(), &apigatewaymanagementapi.PostToConnectionInput{
		ConnectionId: aws.String("connection-id"), // Get from connection table
		Data:         messageBytes,
	})

	if err != nil {
		log.Printf("failed to update dashboard: %v", err)
	}
}
```

## 5. Deployment with SAM

### `template.yaml`
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: Booking Service with Go

Globals:
  Function:
    Runtime: go1.x
    Timeout: 15
    MemorySize: 512
    Environment:
      Variables:
        AWS_REGION: !Ref AWS::Region

Resources:
  BookingsTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: BookingsTable
      AttributeDefinitions:
        - AttributeName: bookingId
          AttributeType: S
        - AttributeName: userId
          AttributeType: S
      KeySchema:
        - AttributeName: bookingId
          KeyType: HASH
      GlobalSecondaryIndexes:
        - IndexName: UserIndex
          KeySchema:
            - AttributeName: userId
              KeyType: HASH
          Projection:
            ProjectionType: ALL
      BillingMode: PAY_PER_REQUEST

  CreateBookingFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: bootstrap
      CodeUri: .
      Policies:
        - DynamoDBCrudPolicy:
            TableName: !Ref BookingsTable
        - SNSPublishMessagePolicy:
            TopicName: BookingNotifications
      Events:
        HttpApiEvent:
          Type: HttpApi
          Properties:
            Path: /bookings
            Method: post

  BookingNotificationTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: BookingNotifications
```

## 6. Build and Deployment

1. **Build your Go Lambda functions**:
```bash
GOOS=linux GOARCH=amd64 go build -o bootstrap main.go
```

2. **Package and deploy with SAM**:
```bash
sam build
sam deploy --guided
```

## Key Advantages of Go Implementation

1. **Faster cold starts** compared to interpreted languages
2. **Better performance** for CPU-intensive operations
3. **Strong typing** reduces runtime errors
4. **Smaller deployment packages**
5. **Better memory efficiency**

## Next Steps

1. Add validation middleware
2. Implement idempotency keys
3. Add integration tests with AWS SDK mocks
4. Set up CI/CD pipeline

Would you like me to elaborate on any specific part of this Go implementation or add any additional components like payment processing or calendar integration?
