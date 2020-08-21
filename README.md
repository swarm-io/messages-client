# Swarm messages grpc client for publishing and subscribing to swarm pipelines

## Publish Example
```go
package main

import (
	"context"
	"crypto/tls"
	"log"

	"github.com/swarm-io/messages-client/proto/messages"
	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials"
	"google.golang.org/grpc/metadata"
)

func main() {
	var options []grpc.DialOption
	// Use TLS and verify the server certificate
	options = append(options, grpc.WithTransportCredentials(credentials.NewTLS(&tls.Config{InsecureSkipVerify: false})))
	options = append(options, grpc.FailOnNonTempDialError(true))
	// Dial to your account's api at {your-account-uuid}.api.swarmiolabs.com:8080
	connection, err := grpc.Dial("b7c65359-559d-492e-b9f1-e6b344ceec5d.api.swarmiolabs.com:8080", options...)
	if err != nil {
		log.Fatalf("fail to dial: %v", err)
	} else {
		defer connection.Close()
		// Create a messagesClient
		messagesClient := messages.NewMessagesClient(connection)
		publishContext := context.Background()
		// Add an api key to the context for authentication.
		publishContext = metadata.AppendToOutgoingContext(publishContext, "x-api-key", "9cb10288-8006-4092-b32f-8dc7d9c358f7")
		// Use the messagesClient to subscribe to a pipeline, this example specifies a group for horizontal scalability
		result, err := messagesClient.PublishMessage(
			publishContext,
			&messages.PublishMessageParameters{
				PipelineId: "pipeline-33abca88-eab5-4b29-9558-5269dcbd10b1",
				Message:    []byte(`{"fieldOne": 1, "fieldTwo": "two"}`)},
		)
		if err != nil {
			log.Printf("Error publishing message\n%s", err)
		} else {
			log.Printf("Got result\n error: %t, errorMessage: %s", result.Error, result.ErrorMessage)
		}
	}
}
```

## Subscribe Example
```go
package main

import (
	"context"
	"crypto/tls"
	"io"
	"log"

	"github.com/swarm-io/messages-client/proto/messages"
	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials"
	"google.golang.org/grpc/metadata"
)

func main() {
	var options []grpc.DialOption
	// Use TLS and verify the server certificate
	options = append(options, grpc.WithTransportCredentials(credentials.NewTLS(&tls.Config{InsecureSkipVerify: false})))
	options = append(options, grpc.FailOnNonTempDialError(true))
	// Connect to your account's api at {your-account-uuid}.api.swarmiolabs.com:8080
	connection, err := grpc.Dial("b7c65359-559d-492e-b9f1-e6b344ceec5d.api.swarmiolabs.com:8080", options...)
	if err != nil {
		log.Fatalf("fail to dial: %v", err)
	} else {
		defer connection.Close()
		// Create a messagesClient
		messagesClient := messages.NewMessagesClient(connection)
		subscribeContext := context.Background()
		// Add an api key to the context for authentication.
		subscribeContext = metadata.AppendToOutgoingContext(subscribeContext, "x-api-key", "9cb10288-8006-4092-b32f-8dc7d9c358f7")
		// Use the messagesClient to subscribe to a pipeline, this example specifies a group for horizontal scalability
		stream, err := messagesClient.Subscribe(subscribeContext, &messages.SubscribeParameters{
			PipelineId: "pipeline-33abca88-eab5-4b29-9558-5269dcbd10b1",
			GroupName: "my-subscriber-group",
		})
		if err != nil {
			log.Fatalf("%v.Subscribe(_) = _, %v", messagesClient, err)
		} else {
			// Loop through the stream until it's closed.
			for {
				message, err := stream.Recv()
				if err == io.EOF {
					break
				} else if err != nil {
					log.Fatalf("%v.Subscribe(_) = _, %v", messagesClient, err)
				} else {
					log.Printf("Received message: %s", string(message.Data))
				}
			}
		}
	}
}
```