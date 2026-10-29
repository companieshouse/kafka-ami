# kafka-ami

Packer and Ansible configuration to build the Kafka 4.2 AMI. Kafka runs in KRaft mode (no ZooKeeper).

AMI names use the Kafka 3 repository convention: `kafka-ami-<version>`. For Kafka 4.2, release tags produce names such as `kafka-ami-4.2.2`.

For an architecture overview and KRaft explanation see [KAFKA-4.2-README.md](KAFKA-4.2-README.md).

### Ansible

All Ansible configuration resides in the `./ansible` directory. The Ansible configuration will be called during the provisioning step of the Packer build as defined in `./packer/build.pkr.hcl`.

The playbook installs Kafka 4.2 in KRaft mode, prepares the data volume, and installs Kafdrop.

### Packer Variables

All Packer configuration resides in the `./packer` directory and utilises standard Packer configuration syntax.

The template provides the following variables to control the Packer build and provisioning process.

| Variable                   | Type         | Default                          | Description                                                                                                                                               |
| -------------------------- | ------------ | -------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ami_account_ids            | list(string) | -                                | A list of account IDs that have access to launch the resulting AMI(s).                                                                                    |
| ami_name_prefix            | string       | kafka                            | Prefix used for the name tags of resulting AMIs. The version will be appended to this.                                                                    |
| aws_instance_type          | string       | t3.small                         | AWS EC2 instance type used when building the AMI.                                                                                                         |
| aws_region                 | string       | eu-west-2                        | The region in which the AMI will be built.                                                                                                                |
| aws_source_ami_filter_name | string       | amzn2023-base-\*                 | Source AMI filter string as per the DescribeImages API documentation. If multiple match, the latest image will be used.                                   |
| aws_source_ami_owner_id    | string       | 416670754337                     | The source AMI owner ID. Used in combination with `aws_source_ami_filter_name` to match the source AMI.                                                   |
| aws_subnet_filter_name     | string       | -                                | Subnet filter string as per the DescribeSubnets API documentation. If multiple match, the subnet with the greatest number of IPv4 addresses will be used. |
| configuration_group        | string       | kafka                            | The name of the group to which to add the instance for configuration purposes.                                                                            |
| data_volume_iops           | number       | 3000                             | The baseline IOPS for the data EBS volume; 3000 is the gp3 default.                                                                                       |
| data_volume_size_gib       | number       | 20                               | The EC2 instance data volume size in Gibibytes (GiB).                                                                                                     |
| data_volume_throughput     | number       | 125                              | The throughput, in MiB/s, for the data EBS volume; 125 is the gp3 default.                                                                                |
| force_delete_snapshot      | bool         | false                            | Automatically delete snapshots associated with AMIs deregistered by `force_deregister`.                                                                   |
| force_deregister           | bool         | false                            | Deregister an existing AMI if one with the same name exists.                                                                                              |
| kms_key_id                 | string       | alias/packer-builders-kms        | The KMS key ID or alias used to encrypt the AMI EBS volumes.                                                                                              |
| playbook_file_path         | string       | ../ansible/playbook.yml          | Relative path to the Ansible playbook file.                                                                                                               |
| root_volume_iops           | number       | 3000                             | The baseline IOPS for the root EBS volume; 3000 is the gp3 default.                                                                                       |
| root_volume_size_gib       | number       | 20                               | The EC2 instance root volume size in Gibibytes (GiB).                                                                                                     |
| root_volume_throughput     | number       | 125                              | The throughput, in MiB/s, for the root EBS volume; 125 is the gp3 default.                                                                                |
| ssh_clear_authorized_keys  | bool         | true                             | Defines whether the authorized_keys file should be cleared, post-build.                                                                                   |
| ssh_private_key_file       | string       | /home/packer/.ssh/packer-builder | The path to the common Packer builder private SSH key.                                                                                                    |
| ssh_username               | string       | ec2-user                         | The username Packer will use when connecting with SSH.                                                                                                    |
| version                    | string       | -                                | Semantic version number for the AMI. Will be automatically appended to `ami_name_prefix` to tag the resulting AMI and snapshots.                          |
