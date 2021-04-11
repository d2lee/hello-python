FROM docker:dind

RUN apk upgrade && apk add -q curl jq py3-pip python3-dev && pip3 install --upgrade awscli
RUN curl -sS -o aws-iam-authenticator https://amazon-eks.s3-us-west-2.amazonaws.com/1.10.3/2018-07-26/bin/linux/amd64/aws-iam-authenticator
RUN curl -sS -o kubectl https://amazon-eks.s3-us-west-2.amazonaws.com/1.14.6/2019-08-22/bin/linux/amd64/kubectl
RUN chmod +x ./kubectl ./aws-iam-authenticator
