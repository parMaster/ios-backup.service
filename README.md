# ios-backup.service

Automatic iPhone and iPad backups to a Linux machine, started when the device is plugged in by USB.

```
plug in -> udev sees an Apple device -> starts ios-backup.service -> ios-backup runs idevicebackup2
```

The backups use the same format as Finder backups on a Mac, so a Mac can restore from them.

## Files

| File | Installed to | Purpose |
|---|---|---|
| `ios-backup` | `/usr/local/bin/ios-backup` | Waits for the device, then backs up each connected device |
| `ios-backup.service` | `/etc/systemd/system/ios-backup.service` | Runs the script as a normal user |
| `90-ios-backup.rules` | `/etc/udev/rules.d/90-ios-backup.rules` | Starts the service on plug-in |

## Requirements

- A Linux machine with systemd and udev. Written for Ubuntu 24.04.
- `usbmuxd`, which talks to iOS devices over USB.
- `libimobiledevice-utils`, which provides `idevice_id`, `idevicepair` and `idevicebackup2`.
- Enough free space for each device. A backup is about the size of the used storage on the device. Photos count, unless they live in iCloud Photos.

## Setup from scratch

### 1. Install the packages

```sh
sudo apt install usbmuxd libimobiledevice-utils
```

### 2. Create the backup folder

Pick a folder on a disk with enough space. The examples use `/data/_backups/ios`.

```sh
mkdir -p /data/_backups/ios
```

### 3. Pair each device once

Do this for every iPhone and iPad, by hand, before the automatic backup can work.

1. Plug the device in and unlock it.
2. Run `idevicepair pair`.
3. Tap Trust on the device and enter its passcode.
4. Run `idevice_id -l`. The device ID should appear.

### 4. Turn on backup encryption

```sh
idevicebackup2 -i encryption on
```

iOS asks for the passcode, then `idevicebackup2` asks for a new backup password.

Encryption is worth it. Without it, the backup leaves out saved passwords, Wi-Fi settings and Health data. The automatic backup skips a device with encryption off, and logs a warning.

Store the backup password safely. Nothing can restore the backup without it.

### 5. Run the first backup by hand

```sh
idevicebackup2 backup /data/_backups/ios
```

The first backup is a full copy, so it takes a while. The command ends by itself when the backup is complete. The device may lock during the backup, and that is fine. Keep the cable connected.

The tool creates one folder per device, named after the device ID. Later backups copy only changes.

### 6. Set the user and folder

Edit `ios-backup.service` before you install it:

- `User=` must be the user that paired the devices and owns the backup folder.
- `BACKUP_DIR=` must be the folder from step 2.

### 7. Install the files

```sh
sudo install -m 755 ios-backup /usr/local/bin/ios-backup
sudo install -m 644 ios-backup.service /etc/systemd/system/ios-backup.service
sudo install -m 644 90-ios-backup.rules /etc/udev/rules.d/90-ios-backup.rules
sudo systemctl daemon-reload
sudo udevadm control --reload
```

The service is not enabled with `systemctl enable`. The udev rule starts it on plug-in.

### 8. Test it

With a device plugged in, start the service by hand and watch its log:

```sh
sudo systemctl start ios-backup
journalctl -u ios-backup -f
```

Then unplug the device, plug it in again, and check that the log shows a new run.

## Daily use

Plug the device in. The backup starts by itself.

- iOS may ask for the passcode when a backup starts. If so, unlock the device once after you plug it in.
- The script skips a device that finished a backup in the last 60 minutes. Change that with an `Environment=SKIP_MINUTES=...` line in the service file.
- An interrupted backup is not skipped. The next plug-in runs it again.

Check past runs with:

```sh
journalctl -u ios-backup
```

## Restore

On a Mac:

1. Copy the device folder, for example `/data/_backups/ios/<device id>`, into `~/Library/Application Support/MobileSync/Backup/`.
2. Plug the device into the Mac and open it in Finder.
3. Click Restore Backup, then pick the backup and enter the backup password.

On Linux, `idevicebackup2 restore /data/_backups/ios` restores directly. Run `idevicebackup2 --help` for its options.

## How it works

- The `usbmuxd` package ships a udev rule, `39-usbmuxd.rules`, that starts `usbmuxd.service` for Apple devices.
- `90-ios-backup.rules` matches the same devices and adds `ios-backup.service` to the services that systemd starts. It uses `+=`, so the `usbmuxd.service` entry stays.
- The rule fires on both the `add` and `bind` events. The 60-minute skip stops the second event from running a second backup.
- The script waits up to 60 seconds for `idevice_id -l` to list the device, because `usbmuxd` needs a moment after plug-in.
- Before each backup, the script reads `WillEncrypt` with `ideviceinfo`. It skips the device if encryption is off.
- The service runs as a normal user. The `usbmuxd` socket is open to all users, so no root access is needed for the backup.

## Troubleshooting

| Symptom | Cause and fix |
|---|---|
| `no iOS device found` in the log | The device is not paired, or it is locked on a "Trust this computer?" prompt. Repeat step 3. |
| `Could not connect to lockdownd, error code -19` | The "Trust this computer?" prompt is still waiting on the device. Unlock it, tap Trust, enter the passcode, then plug it in again. |
| `cannot read device info, is the Trust prompt still open?` | Same cause as error -19. The script could not query the device before the backup. |
| `backup encryption is off, skipping` | Step 4 was skipped for this device. Run the `idevicebackup2 -u <device id> -i encryption on` command from the log, then plug the device in again. |
| The service never starts on plug-in | Run `udevadm monitor --property` and plug the device in. Check that `PRODUCT` starts with `5ac/12`. |
| `backup failed` in the log | Run `idevicebackup2 backup /data/_backups/ios` by hand to see the error. |
| A new iOS version breaks the backup | `libimobiledevice` may need a newer version than the distro ships. |
