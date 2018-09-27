---
layout: "aws"
page_title: "AWS IAM Policy Documents with Terraform"
sidebar_current: "docs-aws-guide-iam-policy-documents"
description: |-
  Using Terraform to configure AWS IAM policy documents.
---

# AWS IAM Policy Documents with Terraform

AWS leverages a standard JSON Identity and Access Management (IAM) policy document format across many services to control authorization to resources and API actions. This guide is designed to highlight some examples how Terraform and the AWS provider allow for many configuration styles to implement these policies.

The example policy documents and resources in this guide are for illustrative purposes only. [Full AWS documentation about the IAM policy format](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements.html).

~> **NOTE** Some AWS services only allow a subset of the policy elements or policy variables. For more information, see the AWS User Guide for the service you are configuring.

<!-- TOC depthFrom:2 -->

- [Escaping IAM Policy Variables](#escaping-iam-policy-variables)
- [Standard String Syntax](#standard-string-syntax)
- [Multiple Line Heredoc Syntax](#multiple-line-heredoc-syntax)
- [aws_iam_policy_document Data Source](#aws_iam_policy_document-data-source)
- [file() Interpolation Function](#file-interpolation-function)
- [template_file Data Source](#template_file-data-source)

<!-- /TOC -->

## Escaping IAM Policy Variables

[IAM policy variables](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_variables.html), e.g. `${aws:username}`, use the same configuration syntax (`${...}`) as Terraform interpolation. When implementing IAM policy documents with these IAM variables, you may receive syntax errors from Terraform. You can escape the dollar character within your Terraform configration to prevent the error, e.g. `$${aws:username}`.

## Standard String Syntax

Single line IAM policy documents can be constructed with regular string syntax. Interpolation is available within the string if necessary. Since the double quotes within the IAM policy JSON conflict with Terraform's double quotes for declaring a string, they need to be escaped (`\"`).

For example:

```hcl
resource "aws_iam_policy" "example" {
  # ... other configuration ...

  policy = "{\"Version\": \"2012-10-17\", \"Statement\": {\"Effect\": \"Allow\", \"Action\": \"*\", \"Resource\": \"*\"}}"
}
```

## Multiple Line Heredoc Syntax

Terraform's heredoc syntax, compared to the standard string syntax, allows easier formatting across multiple lines and does not require escaping double quotes. Interpolation is available within the heredoc string if necessary.

For example:

```hcl
resource "aws_iam_policy" "example" {
  # ... other configuration ...
  policy = <<POLICY
{
  "Version": "2012-10-17",
  "Statement": {
    "Effect": "Allow",
    "Action": "*",
    "Resource": "*"
  }
}
POLICY
}
```

## aws_iam_policy_document Data Source

The AWS provider implements a highly customizable [`aws_iam_policy_document` data source](/docs/providers/aws/d/iam_policy_document.html) to create IAM policy documents. A few benefits include:

- Native Terraform configuration - no need to worry about JSON formatting or syntax
- Policy layering - create policy documents that combine and/or overwrite other policy documents
- Built-in policy error checking

Example:

```hcl
data "aws_iam_policy_document" "example" {
  statement {
    actions   = ["*"]
    resources = ["*"]
  }
}

resource "aws_iam_policy" "example" {
  # ... other configuration ...

  policy = "${data.aws_iam_policy_document.example.json}"
}
```

## file() Interpolation Function

To decouple the IAM policy JSON from the Terraform configuration, Terraform has a built-in [`file()` interpolation function](/docs/configuration/interpolation.html#file-path-), which can read the contents of a local file into the configuration. Interpolation is _not_ available when using the `file()` function by itself.

For example, in a file called `policy.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": {
    "Effect": "Allow",
    "Action": "*",
    "Resource": "*"
  }
}
```

Then `policy.json` contents can be read into the Terraform configuration via:

```hcl
resource "aws_iam_policy" "example" {
  # ... other configuration ...

  policy = "${file("policy.json")}"
}
```

## template_file Data Source

To enable interpolation in decoupled files, the [`template_file` data source](/docs/providers/template/d/file.html) is available.

For example, in a file called `policy.json.tpl`:

```json
{
  "Version": "2012-10-17",
  "Statement": {
    "Effect": "Allow",
    "Action": "*",
    "Resource": "${resource}"
  }
}
```

The contents can be read and interpolated into the Terraform configuration via:

```hcl
data "template_file" "example" {
  template = "${file("policy.json.tpl")}"

  vars {
    resource = "${aws_vpc.example.arn}"
  }
}

resource "aws_iam_policy" "example" {
  # ... other configuration ...

  policy = "${data.template_file.example.rendered}"
}
```
