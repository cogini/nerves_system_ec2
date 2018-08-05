# Nerves System for Amazon EC2

This is a [Nerves](https://nerves-project.org/) system which runs in Amazon
EC2. 

It is based on [nerves_system_x86_64](https://github.com/nerves-project/nerves_system_x86_64),
adding the drivers needed for EC2 to the kernel and configuring the boot
process for the EC2 environment.

# Using

In addition to this base system, you wil need to initialize your project.
[nerves_init_ec2](https://github.com/cogini/nerves_init_ec2) brings up the
base system, similar to [nerves_init_gadget](https://github.com/nerves-project/nerves_init_gadget).
AWS provides [EC2 instance metadata](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html)
to the running system, accessed via HTTP calls to a special IP address.

`nerves_init_ec2` uses this information at runtime to configure the instance.
The most important part is configuring the ssh console to use the SSH
[key pair](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html)
to access the system remotely.

[hello_nerves_ec2](https://github.com/cogini/hello_nerves_ec2) is an example
project which uses `nerves_system_ec2` and `nerves_init_ec2`.

See the blog post [Running Nerves on Amazon EC2](https://www.cogini.com/blog/running-nerves-on-amazon-ec2/)
for more details. 

# Creating nerves_system_ec2

I basically followed the [Nerves documentation for customizing the
system](https://hexdocs.pm/nerves/systems.html#customizing-your-own-nerves-system).

## Make a copy of nerves_system_x86_64 and modify it

```shell
git clone https://github.com/nerves-project/nerves_system_x86_64 nerves_system_ec2
cd nerves_system_ec2/
git remote rename origin upstream
git remote add origin git@github.com:cogini/nerves_system_ec2.git
git push origin master
```

## Configure the Nerves system

```shell
mix nerves.system.shell

make menuconfig
make savedefconfig

make linux-menuconfig
make linux-update-defconfig
```

I used same kernel config as for my [minimal EC2 system with Buildroot](https://github.com/cogini/buildroot_ec2).

## Modify the grub.cfg config

The kernel options are the same as
[nerves_system_x86_64](https://github.com/nerves-project/nerves_system_x86_64),
with a few additions.

Since we can't manually respond to a panic, we just reboot.

    set cloud_opts=panic=1 boot.panic_on_fail

Configure hardware options

    # https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/nvme-ebs-volumes.html#timeout-nvme-ebs-volumes
    set hardware_opts=nvme.io_timeout=4294967295

Set up a serial console, allowing output to be captured in text with "Actions |
Instance Settings | Get System Log" or the AWS CLI command `aws ec2 get-console-output`.

    set console_opts=console=tty1 console=ttyS0

This is the resulting kernel command

    linux (hd0,msdos2)/boot/bzImage root=PARTUUID=04030201-02 rootwait $console_opts $cloud_opts $hardware_opts

## Modify /etc/erlinit.config

Use the serial console:

    -c ttyS0
    -s "/usr/bin/nbtty"

## Modify fwup.conf/fwup-revert.conf

Reduce the size of the user filesystem to match the 1GB volume. In the cloud, we should not be storing
data on the instance, everything should be in S3 or a database.

    define(APP_PART_COUNT, 1013248)
