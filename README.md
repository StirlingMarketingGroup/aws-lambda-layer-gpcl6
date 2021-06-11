# AWS Lambda Layer for Amazon Linux 1 (Golang) Environments for GhostPCL/gpcl6

This repo fixes the following errors while using GhostPCL in AWS Lambda functions written in Golang.

> ./gpcl6-9540-linux-x86_64: /lib64/libc.so.6: version `GLIBC_2.22' not found (required by ./gpcl6-9540-linux-x86_64)

# About

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

## Compiling/Rebuilding This Repo using Amazon Linux

There's no need for you to recompile these libraries, since it's already compiled in this repo, but just in case you want to compile them yourself, then below is a complete list of the steps.

All that's needed to begin is an Amazon Linux 1 EC2 instance, as this is what the Golang Lambda function's runtime is. I used this AMI https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#Images:visibility=public-images;imageId=ami-0aad1231da126e455;sort=name

You might want to use a fast EC2 type for this, since the `make` commands are very slow. Although you will only need this EC2 for an hour or so which won't cost that much, you can also request a spot instance to keep the cost even lower.

### glibc 2.22

```sh
sudo su
cd ~

yum groupinstall 'Development Tools' -y

yum install wget -y

# download the specific version of glibc we're looking for
wget https://ftp.gnu.org/gnu/libc/glibc-2.22.tar.gz

tar -xvzf glibc-2.22.tar.gz

cd glibc-2.22

# no clue what "eag" means, but saw it here https://access.redhat.com/discussions/3244811#comment-1885041
mkdir eag
cd eag

# by default this fails from warnings about "else" brackets,
# so disable that, and also make sure we compile to a custom folder
# using the /opt folder is VERY important, since it's where Lambda Layers'
# files are placed
../configure --prefix=/opt/glibc-2.22 --disable-werror

# sit back for this one
make

make install
```

### gpcl6

```sh
cd ~

wget https://github.com/ArtifexSoftware/ghostpdl-downloads/releases/download/gs9540/ghostpdl-9.54.0.tar.gz

tar -xvzf ghostpdl-9.54.0.tar.gz

cd ghostpdl-9.54.0

# the C++ flags are to make sure we compile while linking against
# our custom glibc version

# tesseract gives errors while compiling, and since
# it has to do with OCR or something we'll just disable it
CXXFLAGS=-'-Wl,--rpath=/opt/glibc-2.22/lib -Wl,--dynamic-linker=/opt/glibc-2.22/lib/ld-linux-x86-64.so.2' ./configure --without-tesseract

# this ones also going to take a while
make

# this should output the gpcl6 info if everything worked
bin/gpcl6
```

# Making The Layer

Everything in the Layer zip gets placed into the `/opt` folder within the Lambda function. With that, we know we want our `glibc-2.22` at the root so that it ends up in the same place our custom `gpcl6` knows to look for it's libraries.

Additionally, everything in the Layer zip's `/bin` directory will be in the Lambda's PATH, so we want our `gpcl6` binary to be in that folder. The file structure should look like this:

```
gpcl6.zip
├── bin
│   ├── gpcl6
└── glibc-2.22
    ├── bin
    ├── etc
    ├── include
    ├── lib
    ├── libexec
    ├── sbin
    ├── share
    └── var

```
