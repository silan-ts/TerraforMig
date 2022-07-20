# TerraforMig <!-- omit in toc -->

>*The missing Terraform state migration tool*

- [Installation](#installation)
- [1. Goal](#1-goal)
- [2. Motivation](#2-motivation)
- [3. Prerequisites](#3-prerequisites)
- [4. Basic Directions](#4-basic-directions)
- [5. CLI](#5-cli)
  - [5.1. Usage](#51-usage)
  - [5.2. Commands](#52-commands)
  - [5.3. Options](#53-options)
- [6. Rollback](#6-rollback)
  - [6.1 Directions](#61-directions)
- [7. Examples](#7-examples)
  - [7.1 Example 1](#71-example-1)
- [8. Testing](#8-testing)
- [9. Debugging](#9-debugging)

## Installation

1. Download the shell script
2. Give executable permissions to the script
3. (Optional) Rename the script to `terraformig` without the `.sh` suffix or create a softlink without the suffix
4. Move the script to a location in your $PATH or add it to your path
   - It's recommended to place this script at the same location as your terraform binary since it is reliant on it

Here's a simple script you can run to install and set up this tool.

```sh
terraform_path="$(which terraform)"
dest_path="$(readlink -f $terraform_path | sed -e 's#/terraform$##')"
curl https://raw.githubusercontent.com/TeraSky-OSS/TerraforMig/master/terraformig.sh --output $dest_path/terraformig
chmod +x $dest_path/terraformig.sh
cmd_path="$(echo $terraform_path | sed -e 's#/terraform$##')"
ln -s $dest_path/terraformig.sh $cmd_path/terraformig
```

## 1. Goal

This project provides a migration tool to move any number of resources from one statefile to another (including remote backends).

## 2. Motivation
  
- There are a few open feature requests related to this usage:
  - [23580](https://github.com/hashicorp/terraform/issues/23580)
  - [21796](https://github.com/hashicorp/terraform/issues/21796)
- While some improvements / bug fixes have been applied for the state management, there does not seem to be any effort on providing a simplified and more robust state migration (through Terraform 0.14.0 __*__)
  >__*__ Last checked on 2020-11-11

## 3. Prerequisites
  
- Shell terminal
- Terraform version 12.13+ (Untested on earlier versions, but may work).
- jq version >= jq-1.5-1-a5b5cbe (Untested on earlier versions, but may work).

## 4. Basic Directions

1. Install the terraformig cli tool. See [installation](#installation).
2. Move the Terraform code that defines the resources you wish to move (including modules) to the Target terraform directory.
3. Switch your current working directory to the source terraform directory from which you are exporting resources.
4. Run the `terraformig plan` command
5. If you are satisfied with the planned operations, run `terraformig apply`. See [CLI](#5-cli) for more information regarding the commands.
6. (Optional) In case, you wish to rollback to a previous state, follow the instructions at [Rollback](#6-rollback).

## 5. CLI

### 5.1. Usage

`terraformig [options] <subcommand> <dest>`

| __Argument__ | __Description__ | __Requirement__ |
|---|---|---|
| \[options] | Command-line flags. List of available flags below. | Optional |
| \<command> | Command to run. List of available commands below. | Required |
| [src path] | Path to source terraform directory | Optional (Defaults to current working directory) |
| \<dest> | Path to destination terraform directory | Required |

### 5.2. Commands

| __Command__ | __Description__ |
|---|---|
| apply | Moves resouces/modules between states. |
| plan | Runs migration tool in DRY_RUN mode without modifying states. |
| purge | Deletes backup files created by this tool in both SRC and DEST Terraform directories. __CAUTION__: Only use if you know what you're doing! __NOTE:__ This does not remove "terraform.tfstate.backup" files which are generated by the terraform command.|
| (Not Implemented) rollback | Recovers previous states in both SRC and DEST Terraform directories. |
| help | Show this help output. |
| version | Show the current Terraformig version. |

### 5.3. Options

| __Flag__ | __Description__ | __Notes__ |
|---|---|---|
|-chdir=DIR | Switch to a different working directory before executing the given subcommand. | Defaults to current working directory. |
|-cleanup | Cleans up any backup files at the successful conclusion of this script | __CAUTION__: Only use if you know what you're doing! __NOTE:__ This does not remove "terraform.tfstate.backup" files which are generated by the terraform command.|
|-auto-approve | Skip interactive approval of plan before applying. | |
|-debug | Enables DEBUG mode which prints otherwise hidden output and enables xtrace. | |
|-help | An alias for the "help" subcommand. | |
|-version | An alias for the "version" subcommand. | |

## 6. Rollback

While you can rollback the states to the previous versions, you will also want to make sure that you move back any resource/module configurations if you intend to continue working with a rollbacked state.

>__NOTE:__ If the state migration was successful and you simply need to revert back all or some or the resources/modules, it is recommended to repeat the [Basic Directions](#4-basic-directions) in the reverse direction.
>Assuming there was some corruption and you cannot simply perform a new terraformig state migration, follow the [Rollback Directions](#61-directions).

### 6.1 Directions

1. If you didn't delete your "terraformig.tfstate.backup" file, this should be the previous state before you ran `terraformig apply`.
   1. You can check that backup state file by running the command: `terraform show terraformig.tfstate.backup`
   2. If you don't have that file anymore, you can run `terraform show` on any other tfstate backup files that are present there.
   3. You can compare the state of the backup file with the current state by running `terraform show` without and filename arguments.
2. Once you've located the backup statefile to use, you should make another backup statefile of the current state. Run `terraform state pull > <NEW_BACKUP_FILE>`.
3. Then to rollback the current terraform directory's state, run `terraform state push -force terraformig.tfstate.backup`.
   1. Replace the filename with whichever backup statefile you are using.
   2. __NOTE:__ The reason you must include `-force` is because terraform won't allow you to rollback a state to an earlier version by default. Feel free to try without the flag to see.
4. Repeat this process with the other terraform directory if needed.

## 7. Examples

### 7.1 Example 1

Suppose I have two terraform directories, "A" and "B", and I would like to move a resource from "A" to "B".

A.tf:

```sh
resource "random_integer" "stays" {
  min     = 1
  max     = 10
}
resource "random_integer" "to_be_moved" {
  min     = 1
  max     = 10
}
```

B.tf:

```sh
resource "random_integer" "stays" {
  min     = 1
  max     = 10
}
```

All that needs to be done, is to move the corresponding resource/module blocks to the new terraform location.
In this case, I will move the resource block `random_integer.to_be_moved` to B.tf (which is in a different directory).
It will then look like the following:

A.tf:

```sh
resource "random_integer" "stays" {
  min     = 1
  max     = 10
}
```

B.tf:

```sh
resource "random_integer" "stays" {
  min     = 1
  max     = 10
}
resource "random_integer" "to_be_moved" {
  min     = 1
  max     = 10
}
```

Now I simply `cd` into the terraform directory of "A" (if I'm not already there), and run `terraformig apply`.
It will ask me to supply the new location of the terraform resources/modules that I've moved (or I can supply it in the apply command).
There will be some confirmations that I need to approve, and that's all!

## 8. Testing

A basic test script is located at the root of this repository. You can run as is or you may also add a remote state configuration to either or both of the terraform repos "tests/bar" and "tests/foo".
More robust testing and instruction to come in the future.

## 9. Debugging

- Unless you supply the `-cleanup` flag during the apply command, you will have to remove any "terraformig.tfstate.backup" files generated before reapplying. You can also run the `purge` command.
