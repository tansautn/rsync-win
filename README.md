# rsync-win — Maintained Fork

## Notice

The original upstream repository is no longer maintained — my pull request has been left unreviewed for over half a year without any response from the maintainers.  
I have forked the project and will continue development here.

Releases are now automatically generated every time I create a new tag.  
Check the [Releases](../../releases) page for the latest builds.

## Differences from Upstream

- Autobuild: Continuous integration automatically builds and publishes binaries when a new tag is pushed.
- **Multiple --exclude support**: You can now use multiple --exclude arguments in one command to filter out several patterns simultaneously.

## Installation

1. Download the latest zip file from the [Releases](../../releases) page.
2. Extract the archive. Keep the cygwin64 folder alongside rsync-win.exe.
3. Add the directory containing rsync-win.exe to your PATH.
4. Restart your terminal and try running rsync-win -h.

## Usage

Example: Exclude multiple patterns and copy from a remote to local folder  
rsync-win.exe --bwlimit=2048 -av --exclude='tmp' --exclude='*.log' --progress -s <REMOTE USER>@<REMOTE_MACHINE>:<REMOTE_PATH> -d ./target/
General options and usage:
Rsync for Windows

Allowed formats for <SRC>/<DEST>:
  local: C:/path/to/file
   ssh: [USER@]HOST:/path/to/file (use --ssh-port to specify the port)
 rsync: rsync://[USER@]HOST[:PORT]/path/to/file

Usage: rsync-win.exe [OPTIONS] --src <SRC> --dest <DEST>

Options:
  -i, --identity <IDENTITY>  SSH identity file [default: "C:/Users/<YOUR USER NAME>/.ssh/id_rsa"]
  -v, --verbose
  -q, --quiet
  -c, --checksum
  -a, --archive
  -r, --recursive
      --delete
      --exclude <EXCLUDE>
      --partial
      --progress
      --bwlimit <BWLIMIT>
  -4, --ipv4
  -6, --ipv6
      --ssh-port <SSH_PORT>
  -s, --src <SRC>
  -d, --dest <DEST>
  -h, --help                 Print help
  -V, --version              Print version
## Contributing

Feel free to open issues and pull requests in this repository.  
Since upstream is unresponsive, all feature development, bug fixes, and support happen here.

## License

This project retains the license of the upstream repo. See [LICENSE](./LICENSE) for more.