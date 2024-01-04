# pbad

This script is intended to be used as a cron job to maintain the accuracy of multiple A or AAAA DNS records for a [Porkbun](https://porkbun.com/)-managed domain. External IP address discovery is done via the Porkbun API, a network interface, or a custom command piped to standard input.

This project is inspired by [gandi-automatic-dns](https://github.com/brianreumere/gandi-automatic-dns).

# Prerequisites

[Generate an API key and enable API access on your domain](https://kb.porkbun.com/article/190-getting-started-with-the-porkbun-api).

# Requirements

  * Bourne shell
  * [curl](https://curl.se/) with HTTP/2 support (7.33.0 or newer)

By default, the script will automatically determine your public IP address using the [`ping` endpoint of the Porkbun API](https://porkbun.com/api/json/v3/documentation#Authentication). For most use cases, this default behavior should be fine. If you'd like to determine your public IP address from a network interface (using the `-i` flag) and your OS doesn't include the `ifconfig` command, you should install a package that provides it (commonly `net-tools`). You may also use the `-s` flag to pipe the output of an arbitrary command that determines your IP address to the standard input of `pbad` (e.g., `curl ipinfo.io/ip | pbad -s -a APIKEY,SECRETKEY -d EXAMPLE.com -r "RECORD-NAMES"`.

# Installation

The simplest way to install `pbad` is to [download the latest release](https://github.com/brianreumere/porkbun-automatic-dns/releases/latest) and run the `pbad` script from the extracted directory. You can optionally add the extracted directory to your `PATH` environment variable or copy the `pbad` script to a location that is already included in your `PATH`.

To set up a crontab entry to run `pbad` on a schedule, store your API key and secret key in the `~/.porkbunapi` file separated by a comma (run `chmod 600 ~/.porkbunapi` to make sure no other users have permissions to this file), and then run `crontab -e` and add a line similar to the following (this example will run `pbad` every 15 minutes and update the `@` and `www` records of the domain `example.net`):

```
0,15,30,45 * * * * /home/brian/porkbun-automatic-dns/pbad -d example.net -r "@ www"
```

The [`brianreumere.software` Ansible collection](https://galaxy.ansible.com/ui/repo/published/brianreumere/software/) contains a `porkbun_automatic_dns` role if you want to deploy `pbad` via Ansible.

The Porkbun API's rate limits are poorly documented, but allegedly are about [2 requests per second or 60 requests per minute](https://github.com/cullenmcdermott/terraform-provider-porkbun/issues/23#issuecomment-1366859999). The `pbad` script is hardcoded to sleep for 1 second before each API call. As long as you aren't updating an excessive number of records, you should be able to run `pbad` more frequently than every 15 minutes if needed.

# Command-line usage

Run `pbad` with no options or `pbad -h` to view this usage info from the command line.

```
Usage: pbad [-h] [-6] [-f] [-t] [-e] [-v] [-s] [-i EXTIF] [-p KEYFILE|-a APIKEY,SECRETKEY] [-l TTL] -d EXAMPLE.COM -r "RECORD-NAMES"

-h: Print this usage info and exit
-6: Update AAAA record(s) instead of A record(s)
-f: Overwrite an existing DNS record regardless of IP address or TTL discrepancy
-t: Just print the updates that would be made without actually creating or updating any DNS records
-e: Print debugging information to stdout
-v: Print information to stdout even if an update isn't needed
-s: Use stdin instead of the Porkbun API to determine external IP address

-i EXTIF: The name of your external network interface (optional, if provided uses ifconfig instead of the Porkbun API to determine external IP address)
-p KEYFILE: Path to the file that contains your comma-separated Porkbun API key and secret key (defaults to ~/.porkbunapi)
-a APIKEY,SECRETKEY: Your Porkbun API key and secret key, separated by a comma (optional, loaded from a file if not specified)
-l TTL: Set a custom TTL on records (optional, defaults to 10800)
-d EXAMPLE.COM: The domain to create or update DNS records for (required)
-r "RECORD-NAMES": A space-separated list of the name(s) of the A or AAAA record(s) to update or create (required)
```

# Function syntax

```
rest "verb" "apiEndpoint" "body"
```

The `rest` function can call arbitrary endpoints of [Porkbun's v3 API](https://porkbun.com/api/json/v3/documentation). The function only accepts `POST` as the first argument, and requires a third parameter to use as the body of the `POST` request. Your Porkbun API key and secret key from the command line, the file specified by the `-p` flag, or the `~/.porkbunapi` file are automatically included in requests.
