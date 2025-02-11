# Go

Source code: [github.com/oras-project/oras-go](https://github.com/oras-project/oras-go)

## Introduction

The ORAS Go client library provides the ability to replicate artifacts between different [Targets](../#target).  
Furthermore, the version `v2` is a registry client conforming [image-spec v1.1.0-rc.2](https://github.com/opencontainers/image-spec/releases/tag/v1.1.0-rc2) and [distribution-spec v1.1.0-rc1](https://github.com/opencontainers/distribution-spec/blob/v1.1.0-rc1/spec.md).

Using the ORAS Go client library, you can develop your own registry client:

```sh
myclient push artifacts.example.com/myartifact:1.0 ./mything.thang
```

## Usage

The package `oras.land/oras-go/v2` can quickly be imported in other Go-based tools that
wish to benefit from the ability to store arbitrary content in container registries.

1. Get the  `oras.land/oras-go/v2` package
```sh
go get oras.land/oras-go/v2
```

2. Import and use the `v2` package
```go
import "oras.land/oras-go/v2"
```

3. Run
```sh
go mod tidy
```

**The API documentation and examples are available at [pkg.go.dev](https://pkg.go.dev/oras.land/oras-go/v2).**

## Quick Start

### Push files to a remote repository

```go
package main

import (
    "context"
    "fmt"

    v1 "github.com/opencontainers/image-spec/specs-go/v1"
    "oras.land/oras-go/v2"
    "oras.land/oras-go/v2/content/file"
    "oras.land/oras-go/v2/registry/remote"
    "oras.land/oras-go/v2/registry/remote/auth"
    "oras.land/oras-go/v2/registry/remote/retry"
)

func pushFiles() {
    // 0. Create a file store
    fs, err := file.New("/tmp/")
    if err != nil {
        panic(err)
    }
    defer fs.Close()
    ctx := context.Background()

    // 1. Add files to a file store
    mediaType := "example/file"
    fileNames := []string{"/tmp/myfile"}
    fileDescriptors := make([]v1.Descriptor, 0, len(fileNames))
    for _, name := range fileNames {
        fileDescriptor, err := fs.Add(ctx, name, mediaType, "")
        if err != nil {
            panic(err)
        }
        fileDescriptors = append(fileDescriptors, fileDescriptor)
        fmt.Printf("file descriptor for %s: %v\n", name, fileDescriptor)
    }

    // 2. Pack the files and tag the packed manifest
    // Note:
    // oras.Pack() packs an artifact manifest by default.
    // If the remote repository does not support the artifact manifest media type,
    // try Image manifest instead by specifying oras.PackOptions{PackImageManifest: true}.
    artifactType := "example/files"
    manifestDescriptor, err := oras.Pack(ctx, fs, artifactType, fileDescriptors, oras.PackOptions{})
    if err != nil {
        panic(err)
    }
    fmt.Println("manifest descriptor:", manifestDescriptor)

    tag := "latest"
    if err = fs.Tag(ctx, manifestDescriptor, tag); err != nil {
        panic(err)
    }

    // 3. Connect to a remote repository
    reg := "myregistry.example.com"
    repo, err := remote.NewRepository(reg + "/myrepo")
    if err != nil {
        panic(err)
    }
    // Note: The below code can be omitted if authentication is not required
    repo.Client = &auth.Client{
        Client: retry.DefaultClient,
        Cache:  auth.DefaultCache,
        Credential: auth.StaticCredential(reg, auth.Credential{
            Username: "username",
            Password: "password",
        }),
    }

    // 3. Copy from the file store to the remote repository
    _, err = oras.Copy(ctx, fs, tag, repo, tag, oras.DefaultCopyOptions)
    if err != nil {
        panic(err)
    }
}
```

### Pull files from a remote repository

```go
func pullFiles() {
    // 0. Create a file store
    fs, err := file.New("/tmp/")
    if err != nil {
        panic(err)
    }
    defer fs.Close()

    // 1. Connect to a remote repository
    ctx := context.Background()
    reg := "myregistry.example.com"
    repo, err := remote.NewRepository(reg + "/myrepo")
    if err != nil {
        panic(err)
    }
    // Note: The below code can be omitted if authentication is not required
    repo.Client = &auth.Client{
        Client: retry.DefaultClient,
        Cache:  auth.DefaultCache,
        Credential: auth.StaticCredential(reg, auth.Credential{
            Username: "username",
            Password: "password",
        }),
    }

    // 2. Copy from the remote repository to the file store
    tag := "latest"
    manifestDescriptor, err := oras.Copy(ctx, repo, tag, fs, tag, oras.DefaultCopyOptions)
    if err != nil {
        panic(err)
    }
    fmt.Println("manifest descriptor:", manifestDescriptor)
}
```
