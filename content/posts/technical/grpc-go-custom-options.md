---
title: "Extend grpc methods with custom options in Go"
date: 2024-12-23T17:29:51+07:00
tags: ["go", "grpc", "protobuf"]
---

## Custom method option in grpc go

Recently, I was implementing various apis in Go with gppc/grpc-gateway and a problem arose. Have a look at this example:

```protobuf
message GetFlowersRequest {
}

message GetFlowersResponse {
  repeated Flower flowers = 1;
}

message GetMushroomsRequest {
}

message GetMushroomsResponse {
  repeated Mushroom mushrooms = 1;
}

message Flower {
  string name = 1;
  string color = 2;
}

message Mushroom {
  string name = 1;
  int64 size = 2;
}

service GardenService {
  rpc GetFlowers(GetFlowersRequest) returns (GetFlowersResponse);
  rpc GetMushrooms(GetMushroomsRequest) returns (GetMushroomsResponse);
}
```

In this example, our GardenService has two methods: GetFlowers and GetMushrooms, but not every user can access both methods.  
Everyone can GetFlowers, but only some users can access GetMushrooms, which is determined by the user's tier.  
User's tier information is provided by an upstream service via gRPC Metadata, and we need to check the user's tier before calling GetMushrooms.  
The simplest way to do this is to add some validation logic to our grpc handlers:
```go
func (s *GardenService) GetMushrooms(ctx context.Context, _ *gardenservicev1.GetMushroomsRequest) (*gardenservicev1.GetMushroomsResponse, error) {
	md, ok := metadata.FromIncomingContext(ctx)
	if !ok {
		return nil, status.Error(codes.InvalidArgument, "metadata not found")
	}
	var (
		userTier int
		err     error
	)
	for _, t := range md.Get("tier") {
		userTier, err = strconv.Atoi(t)
		if err != nil {
			return nil, status.Error(codes.InvalidArgument, "invalid tier")
		}
	}
	if userTier < 2 { // only users with tier 2 or above can get mushrooms
		return nil, status.Error(codes.PermissionDenied, "not allowed to get flowers")
	}
    // ... the rest of the method
}
```
This works, but it is quite repetitive. We have to add this validation logic to every method that requires tier validation.  
The better solution is to extract this logic into a UnaryInterceptor (middleware in the grpc world) for our grpc server.

```go
func RequireTierUnaryInterceptor(ctx context.Context, req any, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (any, error) {
	md, ok := metadata.FromIncomingContext(ctx)
	if !ok {
		return nil, status.Error(codes.InvalidArgument, "metadata not found")
	}
	var (
		userTier int
		err     error
	)
	for _, t := range md.Get("tier") {
		userTier, err = strconv.Atoi(t)
		if err != nil {
			return nil, status.Error(codes.InvalidArgument, "invalid tier")
		}
	}
	if userTier < 2 {
		return nil, status.Error(codes.PermissionDenied, "not allowed to get flowers")
	}
	return handler(ctx, req)
}

// register grpc server
func ServerGrpc() {
	...
	server := grpc.NewServer(grpc.UnaryInterceptor(RequireTierUnaryInterceptor))
	...
}
```
This approach is much better, but by adding this interceptor to our grpc server, all of our methods will require user with tier 2 or above, while we only need this for some methods.  
Since grpc doesn't provide a way to add interceptors to specific methods, we need a way to add minimum tier requirements to our methods.

### Extending grpc methods with custom options
We can attach specific information to grpc methods using custom options.   
The documentation for custom options can be found [here](https://protobuf.dev/programming-guides/proto3/#customoptions).
```protobuf
import "google/protobuf/descriptor.proto";

extend google.protobuf.MethodOptions {
  int32 minimum_tier = 50000; // field number 50000 and above are reserved for user-defined options
}
```
Add the method option to our service definition:
```protobuf
service GardenService {
  rpc GetFlowers(GetFlowersRequest) returns (GetFlowersResponse){
    option (api.v1.minimum_tier) = 0;
  };
  rpc GetMushrooms(GetMushroomsRequest) returns (GetMushroomsResponse){
    option (api.v1.minimum_tier) = 2;
  };
}
```
Now we can extract the minimum tier requirement from the method options in our interceptor:
```go
func getMinimumUserTier(fullMethodName string) int {
	const defaultTier = 0
	methodParts := strings.Split(fullMethodName, "/")
	if len(methodParts) != 3 {
		return defaultTier
	}
	serviceName, methodName := methodParts[1], methodParts[2]

	// Find the service descriptor
	serviceDescriptor, err := protoregistry.GlobalFiles.FindDescriptorByName(protoreflect.FullName(serviceName))
	if err != nil {
		return defaultTier
	}

	// Find the method descriptor
	serviceDesc, ok := serviceDescriptor.(protoreflect.ServiceDescriptor)
	if !ok {
		return defaultTier
	}
	methodDesc := serviceDesc.Methods().ByName(protoreflect.Name(methodName))
	if methodDesc == nil {
		return defaultTier
	}
	// Check for the custom option
	opts := methodDesc.Options().(*descriptorpb.MethodOptions)
	if proto.HasExtension(opts, apiv1.E_MinimumTier) {
		return proto.GetExtension(opts, apiv1.E_MinimumTier).(int)
	}

	return defaultTier
}
```
Now we can use this function in our interceptor:
```go
func RequireTierUnaryInterceptor(ctx context.Context, req any, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (any, error) {
    md, ok := metadata.FromIncomingContext(ctx)
    if !ok {
        return nil, status.Error(codes.InvalidArgument, "metadata not found")
    }
    var (
        userTier int
        err      error
    )
    for _, t := range md.Get("tier") {
        userTier, err = strconv.Atoi(t)
        if err != nil {
            return nil, status.Error(codes.InvalidArgument, "invalid tier")
        }
    }
    if userTier < getMinimumUserTier(info.FullMethod) {
        return nil, status.Error(codes.PermissionDenied, "not allowed to get flowers")
    }
    return handler(ctx, req)
}
```

### References:
- https://protobuf.dev/programming-guides/proto3/#customoptions
- full code example: https://github.com/duchng/grpc_method_extension





