# TerraformDevKit

[![Build Status](https://travis-ci.org/vistaprint/TerraformDevKit.svg?branch=master)](https://travis-ci.org/vistaprint/TerraformDevKit) [![Build status](https://ci.appveyor.com/api/projects/status/74s4yd7dmuwg5tmn/branch/master?svg=true)](https://ci.appveyor.com/project/Vistaprint/terraformdevkit/branch/master)

Set of scripts to ease development and testing with [Terraform](https://www.terraform.io/).

The script collection includes support for:

* Managing AWS credentials
* Backing up the state from a failed Terraform execution
* Executing external commands
* Simple configuration management
* Simple reading and writing to AWS DynamoDB
* Multiplatform tools
* Making simple HTTP requests
* Retrying a block of code
* Terraform environment management
* Locally installing Terraform
* Filtering Terraform logging messages

Most of these scripts exist to provide support to a module development and testing environment for Terraform: [TerraformModules](https://github.com/vistaprint/TerraformModules). But, they might be useful for other purposes too.

## Installation

Add this line to your application's Gemfile:

```ruby
gem 'TerraformDevKit'
```

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install TerraformDevKit

## Usage

To use the library simply import it as:

```ruby
require 'TerraformDevKit'
```    

## Managing Terraform Environments

TerraformDevKit provides a set of Rake tasks and Ruby scripts to ease the management of multiple Terraform environments. Three major environment types are supported: `dev`, `test` and `prod`.

There might be many development environments (`dev`), each one with its own name. Development environments use a local Terraform backend. They are intented to be used by developers while adding features to the infrastructure.

Testing (`test`) and production (`prod`) environment use a remote backend. Thus, the Terraform state file is not kept in the local disk, but on S3. This allows multiple developers to easily work on the same infrastructure instance. For safety reasons, operations that affect testing and production environments require manual user input. This is not the case for development environments.

TerraformDevKit expects templated versions (using [Mustache](https://mustache.github.io/)) of the Terraform files. Such files might contain placeholders for several fields such as `Environment`, (AWS) `Region` or (AWS) `Profile`, among others. TerraformDevKit uses the template files to generate the final files that will be consumed by Terraform. As an example, for the production environment, the resulting files are placed in a directory named `envs/prod`.

### Configuration Files

Configuration files must be placed in a directory named `config`. Each environment type requires a different configuration file. Thus, the following three files must be placed in the `config` directory:

* `config-dev.yml`
* `config-test.yml`
* `config-prod.yml`

The first one contains the configuration for **all** the development environments that are created. The other two contain the configuration for the test and production environments, respectively.

A sample configuration files is shown next:

```yaml
terraform-version: 0.11
project-name: my super cool project
aws:
  profile: myprofile
  region: eu-west-1
```

The AWS profile **must not** be specified for test and production accounts, as users are required to manually type the profile name.

### A Minimal Rakefile

```ruby
ROOT_PATH = File.dirname(File.expand_path(__FILE__))

spec = Gem::Specification.find_by_name 'TerraformDevKit'
load "#{spec.gem_dir}/tasks/devkit.rake"

task :custom_test, [:env] do |_, args|
  # Add some tests here
end
```

### Sample Terraform Templates

The following file (`main.tf.mustache`) contains the infrastructure configuration (a single S3 bucket) as well as information related to the AWS provider.

```hcl
locals {
  env    = "{{Environment}}"
}

terraform {
    backend {{#LocalBackend}}"local"{{/LocalBackend}}{{^LocalBackend}}"s3"{{/LocalBackend}} {}
}

provider "aws" {
  profile = "{{Profile}}"
  region  = "{{Region}}"
}

resource "aws_s3_bucket" "raw" {
  bucket = "foo-${local.env}"
  acl    = "private"

{{#LocalBackend}}
  force_destroy = true
{{/LocalBackend}}
{{^LocalBackend}}
  lifecycle {
    prevent_destroy = true
  }
{{/LocalBackend}}
}
```

The config file requires a `project-name` to be set. This project name is then use to generate the S3 bucket and dynamodb lock table required by terraform to mamage remote state. It is then possible to access these variables inside the `main.tf.mustache` file:

```hcl
terraform {
  {
  {{#LocalBackend}}
    backend = "local" {}
  {{/LocalBackend}}
  {{^LocalBackend}}
    backend = "s3" {
      bucket     = "{{ProjectName}}"
      key        = "{{ProjectAcronym}}-{{Environment}}.tfstate"
      lock_table = "{{ProjectAcronym}}-{{Environment}}-lock-table"
      encrypt    = true
      profile    = "{{Profile}}"
      region     = "{{Region}}"
    }
  {{/LocalBackend}}
  }
}
```

### Injecting Additional Variables into Template Files

In addition to the default variables that are passed to Mustache when rendering
a template file, users can provide additional variables. To do so, users must register a procedure that receives the environment as a parameter and returns
a map with the extra variables and their values. An example is shown next:

```ruby
TDK::TerraformConfigManager.register_extra_vars_proc(
  proc do
    { SumoLogicEndpoint: TDK::Configuration.get('sumologic')['endpoint'] }
  end
)
```

### Skipping Module Updates

For safety and correctness reasons, TerraformDevKit requests to update every
module when it executes Terraform. This may take a while, depending on the
number of modules used in the infrastructure.

When users are sure that an update is not necessary, this step can be skipped
by setting the environment variable `TF_DEVKIT_SKIP_MODULE_UPDATE` to `true`.

## Development

After checking out the repo, run `bin/setup` to install dependencies. Then, run `rake spec` to run the tests. You can also run `bin/console` for an interactive prompt that will allow you to experiment.

To install this gem onto your local machine, run `bundle exec rake install`. To release a new version, update the version number in `version.rb`, and then run `bundle exec rake release`, which will create a git tag for the version, push git commits and tags, and push the `.gem` file to [rubygems.org](https://rubygems.org).

## Contributing

Bug reports and pull requests are welcome on GitHub at https://github.com/vistaprint/TerraformDevKit.

## License

The gem is available as open source under the terms of the [Apache License, Version 2.0](https://opensource.org/licenses/Apache-2.0).
