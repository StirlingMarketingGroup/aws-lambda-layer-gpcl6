# AWS Lambda Layer for Amazon Linux 1 (Golang) Environments for GhostPCL/gpcl6

GhostPCL is a tool for converting PCL files to PDF files. You can download GhostPCL here https://www.ghostscript.com/download/gpcldnld.html.

The GhostPCL for Linux works out of the box in modern Linux systems, like Ubuntu 20.04, but the Golang runtime on AWS Lambda is Amazon Linux 1, unlike what appears to be most of the other environments. This version of Amazon Linux does not support the `glibc` version that GhostPCL requires (version 2.22+).

This repo contains a custom compiled `glibc-2.22` and a custom compiled GhostPCL so that it can be run in this older version of Amazon Linux.

## Usage

Adding this repo as a layer to your AWS account is easy!

```sh
# clone this repo
git clone https://github.com/StirlingMarketingGroup/aws-lambda-layer-gpcl6.git

# cd into the repo
cd aws-lambda-layer-gpcl6

# zip all the contents
zip -r gpcl6.zip .

# this Lambda Layer zip is too big to be uploaded through the UI,
# so we will have to upload it to S3 first.
# $BUCKET=my.cool.bucket
aws s3 cp gpcl6.zip s3://$BUCKET/

# create the Lambda Layer
aws lambda publish-layer-version --layer-name gpcl6 --description "AWS Lambda Layer for Amazon Linux 1 (Golang) Environments for GhostPCL/gpcl6" --license-info "MIT" --content S3Bucket=$BUCKET,S3Key=gpcl6.zip --compatible-runtimes go1.x
```

Now you should be able to add this Layer to your Lambda function by scrolling down to the "Layers" section of the Lambda function "Code" tab by, and by clicking
1. Add Layer
2. Custom Layers
3. Choose "gpcl6" from the dropdown
4. Add

`gpcl6` should now be available in the PATH of your Golang Lambda function.

## Example

Below is a fully functional and complete Lambda function that takes a json request like
```json
{"pcl":"TG9yZW0gaXBzdW0gZG9sciBzaXI=..."}
```
and, if there are no errors, returns
```json
{"pdf":"QmFjb24gaXBzdW0gZG9sb3IgYW1ldCBzaG9ydCBsb2luIGplcmt5..."}
```

```go
package main

import (
	"context"
	"os/exec"

	"github.com/aws/aws-lambda-go/lambda"
	"github.com/pkg/errors"
)

type myEvent struct {
	PCL []byte `json:"pcl"`
}

type myResponse struct {
	PDF []byte `json:"pdf"`
}

func handler(ctx context.Context, e myEvent) (myResponse, error) {
	cmd := exec.Command("gpcl6", "-sDEVICE=pdfwrite", "-o", "-", "-")
	stdin, err := cmd.StdinPipe()
	if err != nil {
		return myResponse{}, errors.Wrapf(err, "failed to set stdin for gpcl")
	}

	go func() {
		defer stdin.Close()
		stdin.Write(e.PCL)
	}()

	b, err := cmd.CombinedOutput()
	if err != nil {
		return myResponse{}, errors.Wrapf(err, "failed to execute gpcl: %s", b)
	}

	resp := myResponse{
		PDF: b,
	}

	return resp, nil
}

func main() {
	lambda.Start(handler)
}

```