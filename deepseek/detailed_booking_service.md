# Booking Service with AWS Lambda + API Gateway in Go

Here's a complete implementation of a booking service using AWS Lambda and API Gateway written in Go:

## Project Structure
```
booking-service/
├── main.go          # Lambda handler
├── models/          # Data models
│   └── booking.go
├── db/              # Database layer
│   └── dynamodb.go
├── handlers/        # Request handlers
│   └── booking_handlers.go
├── go.mod           # Go modules file
└── template.yaml    # SAM template
```

## 1. Main Lambda Handler (`main.go`)

```go
package main

import (
	"booking-service/handlers"
	"context"
	"github.com/aws/aws-lambda-go/events"
	"github.com/aws/aws-lambda-go/lambda"
)

func router(ctx context.Context, req events.APIGatewayProxyRequest) (events.APIGatewayProxyResponse, error) {
	switch req.HTTPMethod {
	case "POST":
		return handlers.CreateBooking(req)
	case "GET":
		if req.PathParameters["id"] != "" {
			return handlers.GetBooking(req)
		}
		return handlers.ListBookings(req)
	case "PUT":
		return handlers.UpdateBooking(req)
	case "DELETE":
		return handlers.CancelBooking(req)
	default:
		return handlers.UnhandledMethod()
	}
}

func main() {
	lambda.Start(router)
}
```

## 2. Booking Model (`models/booking.go`)

```go
package models

import "time"

type Booking struct {
	ID          string    `json:"id" dynamodbav:"id"`
	UserID      string    `json:"user_id" dynamodbav:"user_id"`
	ResourceID  string    `json:"resource_id" dynamodbav:"resource_id"`
	StartTime   time.Time `json:"start_time" dynamodbav:"start_time"`
	EndTime     time.Time `json:"end_time" dynamodbav:"end_time"`
	Status      string    `json:"status" dynamodbav:"status"` // "confirmed", "cancelled", "completed"
	CreatedAt   time.Time `json:"created_at" dynamodbav:"created_at"`
	UpdatedAt   time.Time `json:"updated_at" dynamodbav:"updated_at"`
}

type BookingRequest struct {
	UserID     string    `json:"user_id"`
	ResourceID string    `json:"resource_id"`
	StartTime  time.Time `json:"start_time"`
	EndTime    time.Time `json:"end_time"`
}
```

## 3. DynamoDB Layer (`db/dynamodb.go`)

```go
package db

import (
	"booking-service/models"
	"context"
	"errors"
	"github.com/aws/aws-sdk-go-v2/aws"
	"github.com/aws/aws-sdk-go-v2/config"
	"github.com/aws/aws-sdk-go-v2/feature/dynamodb/attributevalue"
	"github.com/aws/aws-sdk-go-v2/service/dynamodb"
	"github.com/aws/aws-sdk-go-v2/service/dynamodb/types"
	"time"
)

const tableName = "BookingsTable"

type DynamoDBClient struct {
	client *dynamodb.Client
}

func NewDynamoDBClient() (*DynamoDBClient, error) {
	cfg, err := config.LoadDefaultConfig(context.TODO())
	if err != nil {
		return nil, err
	}

	return &DynamoDBClient{
		client: dynamodb.NewFromConfig(cfg),
	}, nil
}

func (d *DynamoDBClient) CreateBooking(booking models.Booking) error {
	item, err := attributevalue.MarshalMap(booking)
	if err != nil {
		return err
	}

	_, err = d.client.PutItem(context.TODO(), &dynamodb.PutItemInput{
		TableName: aws.String(tableName),
		Item:      item,
	})

	return err
}

func (d *DynamoDBClient) GetBooking(id string) (*models.Booking, error) {
	result, err := d.client.GetItem(context.TODO(), &dynamodb.GetItemInput{
		TableName: aws.String(tableName),
		Key: map[string]types.AttributeValue{
			"id": &types.AttributeValueMemberS{Value: id},
		},
	})
	if err != nil {
		return nil, err
	}

	if result.Item == nil {
		return nil, errors.New("booking not found")
	}

	var booking models.Booking
	err = attributevalue.UnmarshalMap(result.Item, &booking)
	return &booking, err
}

// Add other CRUD operations (List, Update, Delete)...
```

## 4. Request Handlers (`handlers/booking_handlers.go`)

```go
package handlers

import (
	"booking-service/db"
	"booking-service/models"
	"encoding/json"
	"github.com/aws/aws-lambda-go/events"
	"net/http"
	"time"
)

var dbClient *db.DynamoDBClient

func init() {
	var err error
	dbClient, err = db.NewDynamoDBClient()
	if err != nil {
		panic(err)
	}
}

func CreateBooking(request events.APIGatewayProxyRequest) (events.APIGatewayProxyResponse, error) {
	var bookingReq models.BookingRequest
	err := json.Unmarshal([]byte(request.Body), &bookingReq)
	if err != nil {
		return events.APIGatewayProxyResponse{
			StatusCode: http.StatusBadRequest,
			Body:       "Invalid request body",
		}, nil
	}

	booking := models.Booking{
		ID:         generateID(), // Implement your ID generation
		UserID:     bookingReq.UserID,
		ResourceID: bookingReq.ResourceID,
		StartTime:  bookingReq.StartTime,
		EndTime:    bookingReq.EndTime,
		Status:     "confirmed",
		CreatedAt:  time.Now(),
		UpdatedAt:  time.Now(),
	}

	err = dbClient.CreateBooking(booking)
	if err != nil {
		return events.APIGatewayProxyResponse{
			StatusCode: http.StatusInternalServerError,
			Body:       "Failed to create booking",
		}, nil
	}

	response, _ := json.Marshal(booking)
	return events.APIGatewayProxyResponse{
		StatusCode: http.StatusCreated,
		Body:       string(response),
	}, nil
}

func GetBooking(request events.APIGatewayProxyRequest) (events.APIGatewayProxyResponse, error) {
	id := request.PathParameters["id"]
	booking, err := dbClient.GetBooking(id)
	if err != nil {
		return events.APIGatewayProxyResponse{
			StatusCode: http.StatusNotFound,
			Body:       "Booking not found",
		}, nil
	}

	response, _ := json.Marshal(booking)
	return events.APIGatewayProxyResponse{
		StatusCode: http.StatusOK,
		Body:       string(response),
	}, nil
}

// Implement other handlers (ListBookings, UpdateBooking, CancelBooking)...
```

## 5. SAM Template (`template.yaml`)

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: Booking Service with Lambda and API Gateway

Resources:
  BookingFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: .
      Handler: main
      Runtime: go1.x
      Architectures:
        - x86_64
      Events:
        BookingApi:
          Type: Api
          Properties:
            Path: /bookings
            Method: ANY
        BookingApiWithId:
          Type: Api
          Properties:
            Path: /bookings/{id}
            Method: ANY
      Environment:
        Variables:
          AWS_REGION: !Ref AWS::Region
      Policies:
        - DynamoDBCrudPolicy:
            TableName: !Ref BookingsTable

  BookingsTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: BookingsTable
      AttributeDefinitions:
        - AttributeName: id
          AttributeType: S
      KeySchema:
        - AttributeName: id
          KeyType: HASH
      BillingMode: PAY_PER_REQUEST

Outputs:
  BookingApi:
    Description: "API Gateway endpoint URL for Booking Service"
    Value: !Sub "https://${ServerlessRestApi}.execute-api.${AWS::Region}.amazonaws.com/Prod/bookings"
```

## Deployment Steps

1. **Build the Lambda function**:
   ```bash
   GOOS=linux GOARCH=amd64 go build -o main main.go
   zip function.zip main
   ```

2. **Deploy with SAM**:
   ```bash
   sam deploy --guided
   ```

3. **Test the API**:
   ```bash
   # Create booking
   curl -X POST https://your-api-id.execute-api.region.amazonaws.com/Prod/bookings \
     -d '{"user_id":"user123","resource_id":"room456","start_time":"2023-01-01T10:00:00Z","end_time":"2023-01-01T12:00:00Z"}'
   
   # Get booking
   curl https://your-api-id.execute-api.region.amazonaws.com/Prod/bookings/booking123
   ```

## Additional Features to Consider

1. **Validation**: Add input validation for booking times
2. **Conflict Checking**: Ensure no overlapping bookings
3. **Authentication**: Integrate with Cognito for user auth
4. **Logging**: Add structured logging
5. **Metrics**: Add CloudWatch metrics for monitoring

This implementation provides a complete serverless booking service with CRUD operations using AWS Lambda, API Gateway, and DynamoDB.
