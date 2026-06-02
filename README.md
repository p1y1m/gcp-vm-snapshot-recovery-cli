# gcp-vm-snapshot-recovery-cli

Practical Google Cloud CLI workflow to recreate a Debian Compute Engine VM from a boot disk snapshot and restore Chrome Remote Desktop access.

# Author

Pedro Yanez Melendez

## Overview

This repository documents a real Google Cloud operational workflow focused on VM recovery, disk-level restoration, CLI-based troubleshooting, and remote access recovery.

The process covers the recreation of an existing Debian-based Google Compute Engine virtual machine from a boot disk snapshot using Google Cloud Shell and the `gcloud` CLI. It also includes the validation of Compute Engine output, the correction of a disk-size error returned by Google Cloud, and the recovery of Chrome Remote Desktop access through Linux service troubleshooting.

This is useful for scenarios involving VM recovery, zone migration, infrastructure reconstruction, operational continuity, and cloud engineering troubleshooting.

## Technical Scope

The workflow includes:

* Configuring the active Compute Engine zone in Google Cloud Shell.
* Testing the direct VM move path and handling the removed `gcloud compute instances move` command.
* Creating a boot disk snapshot from an existing Compute Engine VM.
* Recreating a VM from that snapshot in a target zone.
* Handling disk-size validation errors when the requested boot disk is smaller than the source snapshot.
* Validating the recreated VM output from the CLI.
* Repairing a masked or failed Chrome Remote Desktop service.
* Re-registering the recreated Debian VM as a Chrome Remote Desktop host.
* Keeping only publication-safe technical information in the shared record.

## Technologies Used

* Google Cloud Platform
* Google Compute Engine
* Google Cloud Shell
* `gcloud` CLI
* Persistent Disk Snapshots
* Debian Linux
* Bash
* `systemd`
* Chrome Remote Desktop
* Linux package management with `apt`
* Remote host registration flow

## Workflow Summary

The operational flow followed this sequence:

1. Configure the default Compute Engine zone.
2. Test whether the VM can be moved directly across zones.
3. Use a snapshot-based recreation strategy after the direct move command is rejected.
4. Create a boot disk snapshot from the source VM.
5. Attempt VM recreation from the snapshot.
6. Handle the disk-size validation error returned by Google Cloud.
7. Recreate the VM with a valid boot disk size.
8. Validate the recreated VM status and CLI output.
9. Repair Chrome Remote Desktop service state.
10. Register the recreated VM as a remote desktop host.


## Main CLI Flow

### 1. Configure the default Compute Engine zone

```bash
gcloud config set compute/zone us-central1-a
```

Expected confirmation:

```text
Updated property [compute/zone].
```

### 2. Test direct VM movement

```bash
gcloud compute instances move debian-workstation-vm \
  --zone=us-central1-c \
  --destination-zone=us-central1-a
```

The CLI returned that the command is no longer available:

```text
ERROR: (gcloud.compute.instances.move) This command has been removed.
```

This made the snapshot-based recreation path the correct operational approach.

### 3. Create a boot disk snapshot

```bash
PROJECT=$(gcloud config get-value project)

SNAP=boot-snap-recovery-$(date +%Y%m%d-%H%M%S)

gcloud compute snapshots create $SNAP \
  --source-disk=debian-workstation-vm \
  --source-disk-zone=us-central1-c \
  --storage-location=southamerica-west1
```

Expected confirmation:

```text
Creating gce snapshot boot-snap-recovery-20260529-210203...done.
```

### 4. Attempt VM recreation with an invalid boot disk size

```bash
gcloud compute instances create debian-workstation-vm-restored \
  --zone=southamerica-west1-b \
  --machine-type=e2-standard-2 \
  --source-snapshot=projects/$PROJECT/global/snapshots/$SNAP \
  --boot-disk-size=20GB
```

Google Cloud rejected the request because the new boot disk was smaller than the snapshot requirement:

```text
ERROR: Invalid value for field 'resource.disks[0].initializeParams.diskSizeGb': '20'.
Requested disk size cannot be smaller than the snapshot size (25 GB)
```

### 5. Recreate the VM with the corrected boot disk size

```bash
gcloud compute instances create debian-workstation-vm-restored \
  --zone=southamerica-west1-b \
  --machine-type=e2-standard-2 \
  --source-snapshot=boot-snap-recovery-20260529-210203 \
  --boot-disk-size=25GB
```

Expected output:

```text
Created [https://www.googleapis.com/compute/v1/projects/my-gcp-project-8675xx84/zones/southamerica-west1-b/instances/debian-workstation-vm-restored].

NAME: debian-workstation-vm-restored
ZONE: southamerica-west1-b
MACHINE_TYPE: e2-standard-2
STATUS: RUNNING
```

## Chrome Remote Desktop Recovery

After the VM was recreated, Chrome Remote Desktop required service-level recovery.

### 1. Attempt to enable the service

```bash
sudo systemctl unmask chrome-remote-desktop && \
sudo systemctl daemon-reload && \
sudo systemctl enable --now chrome-remote-desktop
```

The service was still masked:

```text
Failed to enable unit: Unit file /lib/systemd/system/chrome-remote-desktop.service is masked.
```

### 2. Repair the service definition

```bash
sudo rm -f /lib/systemd/system/chrome-remote-desktop.service \
  /etc/systemd/system/chrome-remote-desktop.service && \
sudo apt-get install --reinstall -y chrome-remote-desktop && \
sudo systemctl daemon-reload && \
sudo systemctl enable --now chrome-remote-desktop
```

Relevant output:

```text
Setting up chrome-remote-desktop ...
Created symlink /etc/systemd/system/multi-user.target.wants/chrome-remote-desktop.service -> /lib/systemd/system/chrome-remote-desktop.service.
```

### 3. Restart and inspect the service

```bash
sudo systemctl daemon-reload && \
sudo systemctl restart chrome-remote-desktop && \
sudo systemctl status chrome-remote-desktop --no-pager
```

The service definition existed and was enabled, but the daemon still failed:

```text
Active: failed (Result: exit-code)
ExecStart=/opt/google/chrome-remote-desktop/chrome-remote-desktop-host --type=daemon (code=exited, status=100)
```

### 4. Register the VM as a Chrome Remote Desktop host

```bash
DISPLAY= /opt/google/chrome-remote-desktop/start-host \
  --code="4/publication_safe_authorization_code" \
  --redirect-url="https://remotedesktop.google.com/_/oauthredirect" \
  --name=$(hostname)
```

Expected final confirmation:

```text
Host started successfully.
```

## Key Troubleshooting Points

### Removed direct VM move command

The direct VM move command was rejected by the CLI. The practical resolution was to use a boot disk snapshot and recreate the instance from that snapshot.

### Disk-size validation error

The first recreation attempt failed because the requested boot disk size was smaller than the snapshot size. The correction was to create the VM with a boot disk size equal to or greater than the snapshot requirement.

### Chrome Remote Desktop masked service

The remote desktop service could not be enabled because the service file was masked. The recovery path involved removing the broken service files, reinstalling the package, reloading `systemd`, and registering the host again.

## Security and Publication Safety

This repository does not include real secrets, passwords, PINs, OAuth authorization codes, active public IP addresses, or private Google Cloud identifiers.

Before publishing any similar operational record, verify that the following values are not exposed:

* Real Google account or email address
* Real Google Cloud project ID or project number
* Active public IP address
* OAuth authorization code
* Chrome Remote Desktop PIN
* Host configuration ID
* Production VM names
* Internal network information
* Screenshots containing visible secrets or account identifiers
* Document metadata containing personal or project identifiers

## Why This Matters

This workflow demonstrates practical cloud infrastructure recovery using native Google Cloud tooling and Linux administration. It combines Compute Engine operations, persistent disk snapshot recovery, command-line validation, service troubleshooting, and remote access restoration.

The value of the process is not only in the successful VM recreation, but also in the operational decisions made after each CLI response:

* When a command was removed, the process moved to a supported recovery path.
* When the disk size was invalid, the VM creation command was corrected.
* When the remote desktop service failed, the Linux service state was repaired.
* When the daemon still failed, the host registration flow was completed successfully.

## References

* Google Cloud — Move a VM instance between zones
  https://cloud.google.com/compute/docs/instances/moving-instance-across-zones

* Google Cloud — Create snapshots
  https://cloud.google.com/compute/docs/disks/create-snapshots

* Google Cloud — Restore from snapshots
  https://cloud.google.com/compute/docs/disks/restore-snapshot

* Google Cloud — Chrome Remote Desktop on Compute Engine
  https://cloud.google.com/architecture/chrome-desktop-remote-on-compute-engine

## License

This repository is intended for educational and professional portfolio purposes. Adapt the commands carefully before using them in any real Google Cloud environment.
