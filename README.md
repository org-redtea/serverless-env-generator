Serverless Env Generator Plugin
=======

[![License][ico-license]][link-license] [![NPM][ico-npm]][link-npm]

THIS FORK SUPPORTS ONLY SERVERLESS V4

This is a fork of [serverless-env-generator](https://github.com/DieProduktMacher/serverless-env-generator) with more advanced YAML anchor supporting. See extended description for [Commands](#commands) and [YAML File Structure](#yaml-file-structure) and see [Key features of this fork](#key-features-of-this-fork).

This plugin automatically creates a *.env* file during deployment by merging environment variables from one or more YAML files. During runtime these variables can then be loaded into *process.env* using *dotenv*.

For a brief introduction, read our blogpost about [introducing serverless-env-generator](http://www.dieproduktmacher.com/introducing-serverless-env-generator/).


### Key features:

- Support for multi-stage configurations and custom profiles
- Value of environment variables can be encrypted with AWS KMS, allowing teams to manage sensitive information in git.
- By using KMS, access to secrets can be controlled with IAM. We recommend to create one KMS key per serverless-profile, so you can limit access to credentials to deployment privileges.
- During deployment a temporary .env file is created and uploaded to Lambda by merging and decrypting values of your environment YAML files.
- Environment variables can be loaded with *dotenv* at startup in Lambda without delays from KMS.
- Supports *serverless-local-dev-server* and *serverless offline* for local development.

### Key features of this fork:

- Don`t expand merge directives when modifying env file
- Add new commands that work with anchors
- Support for serverless v4

### Notes

Please note that the uploaded *.env* file contains secrets in cleartext. Therefore we recommend to use [Serverless Crypt](https://github.com/marcy-terui/serverless-crypt) for critical secrets. This tool aims to strike a balance between storing secrets in plaintext in Lambda environment variables and having to decrypt them at runtime using KMS.

Furthermore the tool does not support environment variables generated by Serverless. We recommend to set these variables directly in each functions configuration in *serverless.yml*.

When used with *serverless-local-dev-server* your environment variables are directly loaded into *process.env*. No *.env* file is created to make sure that your local development and deployment tasks do not interfere :-)

This package requires node >= 8.0.
Due to the reliance on KMS, encryption is only supported for AWS.

The `.env.local` file in the project root is here only for the tests.

# Table of Contents

- [Requirements](#requirements)
- [Getting Started](#getting-started)
- [Commands](#commands)
- [YAML File Structure](#yaml-file-structure)
- [Usage with the serverless-plugin-webpack](#usage-with-the-serverless-plugin-webpack)
- [Contribute](#contribute)


# Requirements

- node >= 14
- serverless v3 || v4
- [See below for usage with serverless-plugin-webpack](#usage-with-the-serverless-plugin-webpack)


# Getting Started

### 1. Install the plugin and dotenv

```sh
npm install dotenv --save
npm install @redtea/serverless-env-generator --save-dev
```

### 2. Create a key on KMS

See: https://docs.aws.amazon.com/kms/latest/developerguide/create-keys.html

Please make sure to create the KMS key in the same region as your deployment.

For aliases we recommend to use the service name, for administration privileges no user (your AWS account has full permissions by default) and for usage privileges "serverless-admin" to link access permissions to deployment permissions.


### 3. Add the plugin to your serverless configuration file

*serverless.yml* configuration example:

```yaml
provider:
  name: aws
  runtime: nodejs14.x

functions:
  hello:
    handler: handler.hello

# Add @redtea/serverless-env-generator to your plugins:
plugins:
  - '@redtea/serverless-env-generator'

# Plugin config goes into custom:
custom:
  envFiles: #YAML files used to create .env file
    - environment.yml
  envEncryptionKeyId: #KMS Key used for encrypting values
    dev: ${env:AWS_KMS_KEYID} #Key used for development-stage
```

### 4. Add the .env file to your .gitignore

As the generated *.env* file contains the secrets in cleartext,
make sure that it will never be checked into git!

*.gitignore* code example:

```
.env
```


### 5. Add variables to your environment YAML file

Command example:

```sh
serverless env --attribute name --value "This is not a secret"
serverless env --attribute secret_name --value "This is a secret" --encrypt
```

### 6. Write your function

Note that the *.env* file is automatically created when you deploy your function,
so you can just load those variables with dotenv 🎉

Code example:

```js
require('dotenv').config() // Load variables from .env file

module.exports.hello = (event, context, callback) => {
  const response = {
    statusCode: 200,
    body: JSON.stringify({
      message: process.env.secret_name,
      input: event
    })
  }
  callback(null, response)
}
```

### 7. Deploy & test your function

Command example:

```sh
serverless deploy
serverless invoke -f $FUNCTION_NAME
```

Result example:

```
{
    "body": "{\"input\": {}, \"message\": \"This is a secret\"}",
    "statusCode": 200
}
```


# Commands

You can use these commands to modify your YAML environment files.

If no stage is specified the default one as specified in *serverless.yml* is used.

## Viewing environment variables

Use the following commands to read and decrypt variables from your YAML environment files:

### List variables

```sh
serverless env
serverless env --stage $STAGE
```

### View one variable

```sh
serverless env --attribute $NAME
serverless env --attribute $NAME --stage $STAGE

#shorthand:
sls env -a $NAME
sls env -a $NAME -s $STAGE
```

### Decrypt variables

```sh
serverless env --decrypt
serverless env --attribute $NAME --decrypt
serverless env --attribute $NAME --stage $STAGE --decrypt

#shorthand:
sls env -a $NAME --decrypt
sls env -a $NAME -s $STAGE -d
```

## Setting environment variables

Use the following commands to store and encrypt variables in your YAML environment files:

Note that variables are stored to the first file listed in *envFiles*.

### Set a variable

```sh
serverless env --attribute $NAME --value $PLAINTEXT
serverless env --attribute $NAME --value $PLAINTEXT --stage $STAGE

#shorthand:
sls env -a $NAME -v $PLAINTEXT
sls env -a $NAME -v $PLAINTEXT -s $STAGE
```

### Set and encrypt a variable

```sh
serverless env --attribute $NAME --value $PLAINTEXT --encrypt
serverless env --attribute $NAME --value $PLAINTEXT --stage $STAGE --encrypt

#shorthand:
sls env -a $NAME -v $PLAINTEXT -e
sls env -a $NAME -v $PLAINTEXT -s $STAGE -e
```

### Set a variable for an anchor

```sh
serverless env --anchor $NAME --attribute $NAME --value $PLAINTEXT

#shorthand:
sls env -c $NAME -a $NAME -v $PLAINTEXT
```

### Set and encrypt a variable for an anchor

```sh
serverless env --anchor $NAME --attribute $NAME --value $PLAINTEXT --encrypt

#shorthand:
sls env -c $NAME -a $NAME -v $PLAINTEXT -e
```

# YAML File Structure

Environment variables are stored in stage-agnostic YAML files,
which are then merged into a .env file on deployment.

File example:

```yaml
common: &common
    commonFoo: foo

dev: #stage
    <<: *common
    foo: bar #cleartext variable
    bla: crypted:bc89hwnch8hncoaiwjnd... #encrypted variable

production:
    <<: *common
    foo: baz
    bla: crypted:ncibinv0iwokncoiao3d...
```

You can create additional YAML environment files, for example to include variables that are dynamically generated.
Just add them to the *envFiles* in your *serverless.yml*.

# Usage with the `serverless-plugin-webpack`

In case you are also using the [`serverless-plugin-webpack`](https://github.com/goldwasserexchange/serverless-plugin-webpack) there are some caveats:

## 1. Plugin order in `serverless.yml'

You have to place `@redtea/serverless-env-generator` before the `serverless-plugin-webpack` in the `serverless.yml`

```yaml
# serverless.yml
plugins:
  - '@redtea/serverless-env-generator'
  - serverless-plugin-webpack
```

## 2. Additional `dotenv-webpack`

You need to have the [`dotenv-webpack`](https://github.com/mrsteele/dotenv-webpack) plugin installed:

```sh
npm install dotenv-webpack --save-dev
```

and configured:

```javascript
// webpack.config.js
const Dotenv = require('dotenv-webpack')
module.exports = {
  // ...
  plugins: [
    // ...
    new Dotenv()
  ]
}
```

# Contribute
Anyone is more than welcome to contribute to the @redtea/serverless-env-generator plugin. Here just a few things to consider when doing so:

- this project uses yarn as a package manager
- make sure to pass all tests (run *yarn test*)
- you can add your local *@redtea/serverless-env-generator* version to other projects: yarn add --dev file:/../serverless-env-generator

# License & Credits

Licensed under the MIT license.

This fork created and maintained by [Kirill Khoroshilov](https://github.com/Hokid).

Inspired by [serverless-env-generator](https://github.com/DieProduktMacher/serverless-env-generator).

[ico-license]: https://img.shields.io/github/license/org-redtea/serverless-env-generator.svg
[ico-npm]: https://img.shields.io/npm/v/@redtea/serverless-env-generator.svg

[link-license]: ./LICENSE.txt
[link-npm]: https://www.npmjs.com/package/@redtea/serverless-env-generator  

